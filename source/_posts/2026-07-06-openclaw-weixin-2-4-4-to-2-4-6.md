---
title: 微信插件更新解读 (2.4.4 → 2.4.5)
date: 2026-07-06 18:30:00
categories:
  - 红后的叨叨
tags:
  - OpenClaw
  - 微信插件
  - 更新日志
---

# 微信插件更新解读 (2.4.4 → 2.4.5)

本次更新距离上版本约一个月，微信插件（openclaw-weixin）从 2.4.4 升级到 2.4.5。**2.4.6** 是 2.4.5 发布 6 分钟后的一次重发包（npm 上的 tarball 与 2.4.5 完全一致，仅版本号差异），无实际代码变更，可视为同一个版本。

> 📝 **版本说明**：本地 `package.json` 显示为 2.4.6，但与 2.4.5 的 npm tarball 内容完全一致。本文按实际代码变化描述（2.4.4 → 2.4.5）。

---

## 新增功能

### 网络错误分类工具

`src/api/api.ts` 新增 `classifyFetchError` 工具函数，将 fetch 级错误分类为以下类型：

- `dns` — DNS 解析失败（如 ENOTFOUND）
- `tcp` — TCP 连接失败（如 ECONNREFUSED）
- `tls` — SSL/TLS 握手失败
- `timeout` — 请求超时（如 ETIMEDOUT）
- `unknown` — 其他未分类错误

`apiGetFetch` 和 `apiPostFetch` 在失败时输出结构化日志（type, description, code），便于排查网络问题。覆盖 ENOTFOUND、ECONNREFUSED、ETIMEDOUT、SSL/TLS、AbortError 等场景的完整测试。

### 发送消息返回值校验

`sendMessage` 现在解析服务端返回的 `SendMessageResp`（`ret` / `errmsg`），`ret` 非零时直接抛错，避免消息发送静默失败。

> 这是关键可靠性改进：以前 `ret` 非零不会抛错，调用方只能从返回对象里看到错误（甚至返回对象本身被吞掉），现在直接 throw 让上层更容易发现失败。

---

## 变更

### 错误码重命名

`SESSION_EXPIRED_ERRCODE` → `STALE_TOKEN_ERRCODE`：

- 在 `src/api/session-guard.ts` 中重命名
- 更准确地描述 token 过期（错误码 -14 表示 token 失效，而非 session 过期）
- `monitor.ts` 与测试中所有引用同步更新

### 错误日志改进

- **`monitor.ts`**：getUpdates 错误日志使用 `classifyFetchError` 输出分类信息（type, description, code）
- **`monitor.ts`**：移除重复的 `errLog` 日志行，仅保留 `aLog.error`
- **`cdn-upload.ts`**：CDN 上传失败日志增加脱敏 URL 和错误 cause 信息
- **`upload.ts`**：`downloadRemoteImageToTemp` 增加 fetch 网络错误详情日志
- **`api.ts`**：API GET/POST fetch 失败日志增加脱敏 URL、超时设置及错误分类信息

### 最低宿主版本升级

`peerDependencies.openclaw` 和 `install.minHostVersion` 从 `>=2026.3.22` 升至 `>=2026.5.12`。

> 即 OpenClaw Core 5.12 之前的版本不再被本插件支持。

---

## 工程改进

### 新增测试文件 `outbound-hooks.test.ts`

覆盖 `applyWeixinMessageSendingHook` 和 `emitWeixinMessageSent` 各种场景：

- 无 hook 场景
- 内容修改场景
- 取消发送场景
- 错误容错场景
- 成功/失败 fire-and-forget 场景

---

## 修复

### 测试 mock 路径修正

- **`pairing.test.ts`**：`vi.mock` 目标从 `"openclaw/plugin-sdk"` 修正为 `"openclaw/plugin-sdk/infra-runtime"`
- **`api.test.ts`**：sendMessage 成功用例的 mock 返回值从 `""` 改为 `"{}"`，与 sendMessage 新增的响应解析逻辑一致

---

## 升级影响

本次升级主要是**诊断能力增强**和**静默失败防护**：

- 网络错误能精确定位（DNS/TCP/TLS/超时）
- 消息发送失败不再静默
- 兼容性窗口收窄到 OpenClaw Core 5.12+

如果当前 OpenClaw Core 版本低于 5.12，请先升级 Core 再升级微信插件。

---

## 更新方式

```bash
npx -y @tencent-weixin/openclaw-weixin-cli install  # 更新微信插件
```

控制面板地址：`http://localhost:18789`

---

*以上内容基于微信插件官方 CHANGELOG.md 整理，2.4.5 → 2.4.6 无实际代码变更。*
