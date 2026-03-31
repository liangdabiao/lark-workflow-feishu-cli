---
name: lark-workflow-health-diary
version: 1.0.0
description: "健康日记：通过飞书消息发送食物照片，AI识别食物并记录，追踪饮食和身体状态关联，每周生成健康报告。当用户需要'记录饮食'、'健康日记'、'吃了什么'、'身体不舒服'、'健康追踪'时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# 健康日记工作流

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`../lark-shared/SKILL.md`](../lark-shared/SKILL.md)，其中包含认证、权限处理**

## 适用场景

- "帮我记录今天的午餐" / "记录饮食"
- "我今天胃不舒服" / "昨晚没睡好"
- "这周吃了什么？" / "饮食回顾"
- "帮我分析一下饮食和身体状态的关联"
- "生成健康周报"

## 前置条件

仅支持 **user 身份**。执行前确保已授权：

```bash
lark-cli auth login --domain im,base,docs
```

## 工作流总览

```
记录饮食：
  用户发照片/描述 ──► AI 识别食物 ──► base +record-upsert

记录身体状态：
  用户描述状态 ──► base +record-upsert

关联分析：
  base +data-query ──► AI 分析饮食-状态关联

周报生成：
  base +record-list ──► AI 汇总 ──► doc +create

主动提醒：
  base +record-list ──► 识别风险食物 ──► im +messages-send
```

---

## 数据初始化

### Base 表结构

**表 1：饮食记录 (diet_log)**

| 字段名 | 类型 | 说明 |
|--------|------|------|
| 日期 | 日期 | 记录日期 |
| 餐次 | 单选 | 早餐/午餐/晚餐/加餐/夜宵 |
| 时间 | 日期 | 具体时间 |
| 食物列表 | 文本 | 识别出的食物名称（逗号分隔） |
| 食物详情 | 文本 | 每种食物的详细描述和份量 |
| 照片链接 | URL | 飞书中食物照片的链接 |
| 估算热量 | 数字 | 估算的卡路里（可选，AI估算） |
| 备注 | 文本 | 用户补充说明 |
| 创建时间 | 日期 | 记录创建时间 |

**表 2：身体状态 (body_status)**

| 字段名 | 类型 | 说明 |
|--------|------|------|
| 日期 | 日期 | 状态日期 |
| 时间 | 日期 | 具体时间 |
| 状态类型 | 单选 | 胃部不适/头痛/失眠/疲劳/过敏/皮肤/情绪/其他 |
| 严重程度 | 单选 | 轻微/中等/严重 |
| 状态描述 | 文本 | 详细描述 |
| 可能诱因 | 文本 | 用户猜测的原因（如有） |
| 持续时间 | 文本 | 症状持续时长 |
| 缓解方式 | 文本 | 如何缓解的（如有） |
| 创建时间 | 日期 | 记录创建时间 |

**表 3：关联分析 (correlation)**

| 字段名 | 类型 | 说明 |
|--------|------|------|
| 食物 | 文本 | 可疑食物名称 |
| 状态 | 文本 | 关联的身体状态 |
| 出现次数 | 数字 | 该组合出现的次数 |
| 时间差模式 | 文本 | 食用后多久出现症状（如"2-4小时"） |
| 关联强度 | 单选 | 低/中/高 |
| 最近出现日期 | 日期 | 最近一次出现时间 |
| 备注 | 文本 | 分析备注 |

### 初始化命令

```bash
# 注意：+table-create 可能部分成功，需捕获错误后用 +field-create 补字段
# 注意：连续创建字段会触发限流（错误码 800004135），每次调用间隔至少 1 秒
lark-cli base +table-create --name "饮食记录" --fields '[
  {"name":"日期","type":"datetime"},
  {"name":"餐次","type":"select","multiple":false,"options":[{"name":"早餐"},{"name":"午餐"},{"name":"晚餐"},{"name":"加餐"},{"name":"夜宵"}]},
  {"name":"时间","type":"datetime"},
  {"name":"食物列表","type":"text"},
  {"name":"食物详情","type":"text"},
  {"name":"照片链接","type":"link"},
  {"name":"估算热量","type":"number"},
  {"name":"备注","type":"text"},
  {"name":"创建时间","type":"datetime"}
]'
lark-cli base +table-create --name "身体状态" --fields '[
  {"name":"日期","type":"datetime"},
  {"name":"时间","type":"datetime"},
  {"name":"状态类型","type":"select","multiple":false,"options":[{"name":"胃部不适"},{"name":"头痛"},{"name":"失眠"},{"name":"疲劳"},{"name":"过敏"},{"name":"皮肤"},{"name":"情绪"},{"name":"其他"}]},
  {"name":"严重程度","type":"select","multiple":false,"options":[{"name":"轻微"},{"name":"中等"},{"name":"严重"}]},
  {"name":"状态描述","type":"text"},
  {"name":"可能诱因","type":"text"},
  {"name":"持续时间","type":"text"},
  {"name":"缓解方式","type":"text"},
  {"name":"创建时间","type":"datetime"}
]'
lark-cli base +table-create --name "关联分析" --fields '[
  {"name":"食物","type":"text"},
  {"name":"状态","type":"text"},
  {"name":"出现次数","type":"number"},
  {"name":"时间差模式","type":"text"},
  {"name":"关联强度","type":"select","multiple":false,"options":[{"name":"低"},{"name":"中"},{"name":"高"}]},
  {"name":"最近出现日期","type":"datetime"},
  {"name":"备注","type":"text"}
]'
```

---

## 模式一：记录饮食

### 方式 A：照片识别

用户通过飞书消息发送食物照片：

```bash
# 获取图片消息
lark-cli im +messages-resources-download --message-id "<message_id>" --file-key "<file_key>"
```

**AI 处理：**
1. 使用多模态能力识别照片中的食物
2. 提示用户确认或补充："收到，晚餐是肉酱披萨，吃了几片？"
3. 用户回复后，记录到饮食记录表

### 方式 B：文字描述

用户直接描述吃了什么：
- "午餐吃了米饭、红烧肉和青菜"
- "下午喝了一杯拿铁"

**AI 处理：**
1. 解析食物名称
2. 如有模糊描述，追问细节（如"咖啡加了什么？全脂牛奶还是燕麦奶？"）
3. 估算热量（可选）
4. 记录到饮食记录表

### 记录命令

```bash
lark-cli base +record-upsert --table-id "<diet_table_id>" --json '{
  "日期": "2026-03-31",
  "餐次": "午餐",
  "时间": "2026-03-31 12:30:00",
  "食物列表": "米饭,红烧肉,青菜",
  "食物详情": "米饭一碗、红烧肉约200g、炒青菜一份",
  "备注": ""
}'
```

---

## 模式二：记录身体状态

用户描述身体状态：

- "今天胃不舒服" → 记录：状态类型=胃部不适
- "昨晚失眠了，大概凌晨3点才睡着" → 记录：状态类型=失眠
- "吃完午饭头有点晕" → 记录：状态类型=头痛 + 可能诱因=午餐

```bash
lark-cli base +record-upsert --table-id "<body_table_id>" --json '{
  "日期": "2026-03-31",
  "时间": "2026-03-31 14:00:00",
  "状态类型": "胃部不适",
  "严重程度": "中等",
  "状态描述": "午饭后胃有点胀，持续约1小时",
  "可能诱因": "午餐吃的太快"
}'
```

---

## 模式三：关联分析

AI 自动分析饮食和身体状态之间的关联：

### 分析逻辑

1. **时间窗口匹配**：查找身体状态出现前 2-24 小时内的饮食记录
2. **频率统计**：统计每种食物与每种身体状态的共现频率
3. **时间差模式**：分析食物食用到症状出现的时间间隔规律
4. **排除常见因素**：排除普遍出现的食物（如米饭、水），聚焦特异性食物

### 分析命令

```bash
# 获取最近 30 天的数据
lark-cli base +record-list --table-id "<diet_table_id>" --filter '...日期 >= "30天前"'
lark-cli base +record-list --table-id "<body_table_id>" --filter '...日期 >= "30天前"'
```

### AI 分析输出

```
## 饮食-身体状态关联分析（过去 30 天）

### 发现的关联模式
| 食物 | 身体状态 | 出现次数 | 典型时间差 | 关联强度 |
|------|---------|---------|-----------|---------|
| 冰美式 | 胃部不适 | 3次 | 1-2小时 | 高 |
| 辣味火锅 | 失眠 | 2次 | 6-8小时 | 中 |

### 建议
- 冰美式可能与你的胃部不适有较强关联，建议减少摄入或改用热饮
- 辣味火锅后出现失眠，建议避免晚餐食用辛辣食物
```

### 更新关联分析表

```bash
lark-cli base +record-upsert --table-id "<correlation_table_id>" --json '{
  "食物": "冰美式",
  "状态": "胃部不适",
  "出现次数": 3,
  "时间差模式": "1-2小时",
  "关联强度": "高"
}'
```

---

## 模式四：主动提醒

当用户记录身体状态时，AI 主动检查关联分析表：

```bash
lark-cli base +record-list --table-id "<correlation_table_id>" --filter '...关联强度 IN ("高","中")'
```

如果当前饮食包含高风险食物，立即提醒：

```
提醒：你今天喝了冰美式，之前 3 次喝冰美式后 1-2 小时内都出现了胃部不适。
建议：注意观察，如果出现不适请记录。
```

---

## 模式五：周报生成

每周生成健康报告并保存为飞书文档：

### 报告内容

```
## 健康周报（{日期范围}）

### 饮食概览
- 记录天数：N 天
- 总餐次：N 次
- 饮食规律性：{评价}

### 身体状态概览
- 记录不适：N 次
- 主要症状：{列表}
- 严重程度分布：轻微 N 次、中等 N 次、严重 N 次

### 本周新发现
- {新发现的关联模式}

### 趋势对比（vs 上周）
- 饮食多样性：{变化}
- 不适频率：{变化}

### 建议
- {个性化建议}
```

### 生成命令

```bash
lark-cli doc +create --title "健康周报 ({日期范围})" --markdown "<报告内容>"
```

---

## 权限表

| 命令 | 所需 scope |
|------|-----------|
| `im +chat-messages-list` | `im:message:readonly` |
| `im +messages-resources-download` | `im:message.resource:readonly` |
| `im +messages-send` | `im:message:send_as_bot` |
| `base +table-create` | `bitable:app` |
| `base +record-list` | `bitable:app:read` |
| `base +record-upsert` | `bitable:app:write` |
| `base +data-query` | `bitable:app:read` |
| `docs +create` | `docx:document:create` |

## 参考

- [`lark-shared`](../lark-shared/SKILL.md) — 认证、权限（必读）
- [`lark-im`](../lark-im/SKILL.md) — `+chat-messages-list`、`+messages-resources-download`、`+messages-send` 详细用法
- [`lark-base`](../lark-base/SKILL.md) — `+table-create`、`+record-list`、`+record-upsert`、`+data-query` 详细用法
- [`lark-doc`](../lark-doc/SKILL.md) — `+create` 详细用法
