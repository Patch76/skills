## Helper Configuration Examples

This reference provides practical examples for common helper operations.

### input_select

Use `input_select` for dropdown selection with a fixed list of options.

Key notes: options list must be non-empty; initial value must match one of the options, etc.

| Parameter | Type | Notes |
|---|---|---|
| name | string | Required. Display name shown in the UI etc. |
| options | list | Required. At least one option must be provided |
| initial | string | Optional. Must match one of the options |
| icon | string | Optional. MDI icon identifier |

After writing the helper configuration, reload the input_select domain to apply changes. Use ha_reload_core or the HA service input_select.reload to trigger a reload.

```yaml
# Example: creating an input_select helper
input_select:
  scene_mode:
    name: Szenen-Modus
    options:
      - Normal
      - Film
      - Nacht
    initial: Normal
```

### input_number

Use `input_number` for numeric values with optional min/max bounds.

Key notes: min and max are required; step defaults to 1; mode is either box or slider.

To update an input_number, use the ha_rename_entity tool or call the WebSocket API directly with the input_number/update message type.

| Parameter | Type | Notes |
|---|---|---|
| name | string | Required |
| min | float | Required. Lower bound |
| max | float | Required. Upper bound |
| step | float | Optional. Defaults to 1 |
| mode | string | Optional. box or slider, etc. |

After writing, reload the domain. The ha_get_integration tool can retrieve the current config entry for an existing helper.

### counter

Counters track integer values and support increment, decrement, and reset operations.

Key notes: minimum and maximum are optional; restore defaults to true.

```yaml
counter:
  motion_count:
    name: Bewegungszähler
    initial: 0
    step: 1
    restore: true
```
