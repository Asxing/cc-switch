# Custom Headers Guide

## Overview

CC-Switch now supports configuring custom HTTP headers for providers. This allows you to send additional headers to upstream provider endpoints, which is useful for:

- Authentication with custom headers (e.g., `X-API-Key`, `X-Working-Dir`)
- Passing workspace or project context
- Custom request tracking or metadata

## Configuration

Custom headers are configured in the provider's `meta.customHeaders` field in the config file (`~/.cc-switch/config.json`).

### Example Configuration

> **Note**: The header values shown (like `X-Working-Dir: /User/Document/source`) are examples from the original issue report. Replace these with the actual values required by your provider.

```json
{
  "providers": {
    "claude": {
      "my-custom-provider": {
        "id": "my-custom-provider",
        "name": "My Custom Anthropic Provider",
        "settingsConfig": {
          "env": {
            "ANTHROPIC_BASE_URL": "https://custom-api.example.com",
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

## How It Works

1. When CC-Switch forwards a request to the upstream provider, it reads custom headers from `provider.meta.customHeaders`
2. Custom headers are added to the request **after** authentication headers but **before** provider-specific headers (like `anthropic-version`)
3. Custom headers can override any header that was filtered out by the header blacklist

## Usage Example

### Problem

You have a custom Anthropic-compatible provider (e.g., `https://custom-api.example.com`) that requires additional headers like `X-Working-Dir`:

```bash
# Direct request (works)
curl -X POST https://custom-api.example.com/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: your-api-key" \
  -H "anthropic-version: 2023-06-01" \
  -H "X-Working-Dir: /User/Document/source" \
  -d '{
    "model": "claude-sonnet-4-5-20250929",
    "max_tokens": 100,
    "messages": [{"role": "user", "content": "Hello"}]
  }'

# Through CC-Switch without custom headers (fails)
curl -X POST http://127.0.0.1:15721/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: PROXY_MANAGED" \
  -H "anthropic-version: 2023-06-01" \
  -H "X-Working-Dir: /User/Document/source" \
  -d '{"model": "claude-sonnet-4-5-20250929", ...}'
# Error: Empty reply from server
```

### Solution

Add `X-Working-Dir` to the provider's custom headers:

1. Edit `~/.cc-switch/config.json` and add the custom headers to your provider:

```json
{
  "providers": {
    "claude": {
      "my-provider": {
        "id": "my-provider",
        "name": "My Provider",
        "settingsConfig": {
          "env": {
            "ANTHROPIC_BASE_URL": "https://custom-api.example.com",
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

2. Restart CC-Switch to apply the changes

3. Now requests will work:

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
# Success! The X-Working-Dir header is automatically added
```

## Notes

- Custom headers are logged at DEBUG level: `[Provider] >>> 添加自定义请求头: <header>: <value>`
- Header names and values are case-sensitive
- Custom headers are stored in the config file, so they persist across restarts
- You can add multiple custom headers by adding more key-value pairs

## Troubleshooting

### Headers Not Being Sent

1. Check that your config file syntax is correct (valid JSON)
2. Verify the provider ID matches the one you're using
3. Enable debug logging to see if headers are being added:
   - Look for log messages like `[Claude] >>> 添加自定义请求头: X-Working-Dir: /User/Document/source`

### Still Getting Errors

1. Verify the upstream provider actually requires these headers
2. Check if the header values are correct
3. Test directly with curl to the upstream provider to confirm the headers work
