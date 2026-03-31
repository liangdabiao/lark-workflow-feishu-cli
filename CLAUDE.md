# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This directory is a Feishu (Lark) CLI workspace. It uses `lark-cli` to interact with Feishu APIs for messaging, calendar, contacts, documents, spreadsheets, drive, tasks, and more.

## CLI Tool

The primary tool is `lark-cli` (also known as `lark`). Key commands:

- `lark-cli auth login` — Start device-flow OAuth authorization
- `lark-cli --version` / `lark-cli --help` — Check version or get help
- `lark-cli config:*` — Configuration commands

Supported API modules: `im`, `calendar`, `contact`, `docs`, `sheets`, `base`, `drive`, `mail`, `task`, `vc`, `wiki`, `minutes`, `api` (generic API calls).

## Feishu App

The workspace is linked to Feishu app `cli_a94d22643c38dbcd` (configured via the Feishu open platform CLI setup).

## Permissions

`.claude/settings.local.json` allows Claude Code to run `lark-cli` and `lark` commands, check their availability, and read npm global package listings.
