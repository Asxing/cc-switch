# 自定义请求头指南

## 概述

CC-Switch 现在支持为 Provider 配置自定义 HTTP 请求头。这使您可以向上游 Provider 端点发送额外的请求头，适用于：

- 使用自定义请求头进行身份验证（例如 `X-API-Key`、`X-Working-Dir`）
- 传递工作区或项目上下文
- 自定义请求跟踪或元数据

## 配置

自定义请求头在配置文件（`~/.cc-switch/config.json`）中的 Provider 的 `meta.customHeaders` 字段进行配置。

### 配置示例

> **注意**：示例中显示的请求头值（如 `X-Working-Dir: /User/Document/source`）来自原始问题报告。请将这些替换为您的 Provider 实际需要的值。

```json
{
  "providers": {
    "claude": {
      "my-custom-provider": {
        "id": "my-custom-provider",
        "name": "我的自定义 Anthropic Provider",
        "settingsConfig": {
          "env": {
            "ANTHROPIC_BASE_URL": "https://mcli.sankuai.com",
            "ANTHROPIC_AUTH_TOKEN": "your-api-key"
          }
        },
        "meta": {
          "customHeaders": {
            "X-Working-Dir": "/User/Document/source",
            "X-Custom-Header": "custom-value"
          }
        }
      }
    }
  }
}
```

## 工作原理

1. 当 CC-Switch 将请求转发到上游 Provider 时，它会从 `provider.meta.customHeaders` 读取自定义请求头
2. 自定义请求头会在认证请求头**之后**、Provider 特定请求头（如 `anthropic-version`）**之前**添加到请求中
3. 自定义请求头可以覆盖被请求头黑名单过滤掉的任何请求头

## 使用示例

### 问题描述

您有一个自定义的 Anthropic 兼容 Provider（例如 `https://mcli.sankuai.com`），它需要额外的请求头如 `X-Working-Dir`：

```bash
# 直接请求（成功）
curl -X POST https://mcli.sankuai.com/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: your-api-key" \
  -H "anthropic-version: 2023-06-01" \
  -H "X-Working-Dir: /User/Document/source" \
  -d '{
    "model": "claude-sonnet-4-5-20250929",
    "max_tokens": 100,
    "messages": [{"role": "user", "content": "Hello"}]
  }'

# 通过 CC-Switch 但不使用自定义请求头（失败）
curl -X POST http://127.0.0.1:15721/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: PROXY_MANAGED" \
  -H "anthropic-version: 2023-06-01" \
  -H "X-Working-Dir: /User/Document/source" \
  -d '{"model": "claude-sonnet-4-5-20250929", ...}'
# 错误：Empty reply from server
```

### 解决方案

将 `X-Working-Dir` 添加到 Provider 的自定义请求头中：

1. 编辑 `~/.cc-switch/config.json` 并将自定义请求头添加到您的 Provider：

```json
{
  "providers": {
    "claude": {
      "my-provider": {
        "id": "my-provider",
        "name": "我的 Provider",
        "settingsConfig": {
          "env": {
            "ANTHROPIC_BASE_URL": "https://mcli.sankuai.com",
            "ANTHROPIC_AUTH_TOKEN": "your-api-key"
          }
        },
        "meta": {
          "customHeaders": {
            "X-Working-Dir": "/User/Document/source"
          }
        }
      }
    }
  }
}
```

2. 重启 CC-Switch 以应用更改

3. 现在请求将会成功：

```bash
curl -X POST http://127.0.0.1:15721/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: PROXY_MANAGED" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "claude-sonnet-4-5-20250929",
    "max_tokens": 100,
    "messages": [{"role": "user", "content": "Hello"}]
  }'
# 成功！X-Working-Dir 请求头会自动添加
```

## 注意事项

- 自定义请求头会在 DEBUG 日志级别记录：`[Provider] >>> 添加自定义请求头: <header>: <value>`
- 请求头名称和值是区分大小写的
- 自定义请求头存储在配置文件中，因此会在重启后保留
- 您可以通过添加更多键值对来添加多个自定义请求头

## 故障排除

### 请求头未被发送

1. 检查您的配置文件语法是否正确（有效的 JSON）
2. 验证 Provider ID 是否与您正在使用的匹配
3. 启用调试日志以查看请求头是否被添加：
   - 查找类似 `[Claude] >>> 添加自定义请求头: X-Working-Dir: /User/Document/source` 的日志消息

### 仍然出现错误

1. 验证上游 Provider 实际上是否需要这些请求头
2. 检查请求头值是否正确
3. 直接使用 curl 向上游 Provider 测试以确认请求头有效
