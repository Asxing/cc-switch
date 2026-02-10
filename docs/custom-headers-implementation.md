# Custom Headers Feature - Implementation Summary

## Overview
This implementation adds support for configuring custom HTTP headers in provider configurations, enabling users to send additional headers to upstream Anthropic-compatible providers.

## Changes Made

### 1. Provider Data Model (`src-tauri/src/provider.rs`)
- Added `custom_headers: HashMap<String, String>` field to `ProviderMeta` struct
- Configured with serde attributes:
  - `#[serde(rename = "customHeaders")]` - for camelCase JSON serialization
  - `#[serde(default, skip_serializing_if = "HashMap::is_empty")]` - omit when empty
- Added comprehensive tests for serialization/deserialization

### 2. Request Forwarding (`src-tauri/src/proxy/forwarder.rs`)
- Custom headers are applied in the request building pipeline:
  1. After authentication headers (line 654)
  2. Before provider-specific headers like `anthropic-version` (line 667)
- Each header is logged at DEBUG level for troubleshooting
- Headers are added using reqwest's safe `.header()` API

### 3. Tests (`src-tauri/src/proxy/providers/claude.rs`, `src-tauri/src/provider.rs`)
- Unit tests for ProviderMeta serialization/deserialization
- Integration tests for custom headers in Claude adapter
- Tests verify:
  - Correct camelCase naming in JSON
  - Empty collections are not serialized
  - Multiple headers work correctly

### 4. Documentation
- English guide: `docs/custom-headers-guide.md`
- Chinese guide: `docs/custom-headers-guide-zh.md`
- Example configuration: `docs/examples/custom-headers-config.json`
- Includes troubleshooting section and usage examples

## Security Considerations

### ✅ Safe Aspects
1. **Type Safety**: Headers are stored in a `HashMap<String, String>` - no injection risk
2. **API Safety**: Headers are applied via reqwest's safe `.header()` method
3. **Logging**: Headers logged at DEBUG level only - not exposed in production
4. **User Control**: Headers are in user-controlled config file - appropriate for this use case

### ⚠️ Intentional Behaviors
1. **Header Override**: Custom headers CAN override filtered headers - this is by design
   - Users may need to set headers that would normally be filtered
   - This is the intended functionality for custom providers
2. **No Header Validation**: We don't validate header names/values
   - Users are responsible for valid headers
   - Invalid headers will cause upstream provider errors (expected behavior)

## Usage Example

```json
{
  "providers": {
    "claude": {
      "my-provider": {
        "id": "my-provider",
        "name": "My Custom Provider",
        "settingsConfig": {
          "env": {
            "ANTHROPIC_BASE_URL": "https://custom-api.example.com",
            "ANTHROPIC_AUTH_TOKEN": "your-key"
          }
        },
        "meta": {
          "customHeaders": {
            "X-Working-Dir": "/path/to/workspace",
            "X-Project-ID": "project-123"
          }
        }
      }
    }
  }
}
```

## Testing

### Unit Tests
```bash
cd src-tauri
cargo test provider_meta_serializes_custom_headers
cargo test provider_meta_deserializes_custom_headers
cargo test test_custom_headers_in_provider_meta
```

### Manual Testing
1. Add custom headers to provider config
2. Restart CC-Switch
3. Send request to proxy endpoint
4. Verify headers are forwarded to upstream provider
5. Check DEBUG logs for header application confirmation

## Troubleshooting

### Headers Not Being Applied
1. Verify JSON syntax in config file
2. Check provider ID matches active provider
3. Enable DEBUG logging to see header application
4. Restart CC-Switch after config changes

### Upstream Provider Errors
1. Verify header names and values are correct
2. Test directly with curl to upstream provider
3. Check upstream provider documentation for required headers

## Files Modified
- `src-tauri/src/provider.rs` (+60 lines)
- `src-tauri/src/proxy/forwarder.rs` (+8 lines)
- `src-tauri/src/proxy/providers/claude.rs` (+35 lines)
- `docs/custom-headers-guide.md` (new)
- `docs/custom-headers-guide-zh.md` (new)
- `docs/examples/custom-headers-config.json` (new)

## Future Enhancements (Not in Scope)
- UI for editing custom headers (currently JSON-only)
- Header validation/sanitization
- Per-request header override
- Header templates/macros
