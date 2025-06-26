# EnOcean PTM 21xZ/ZE Continuous Trigger Blueprint

## Overview

This enhanced blueprint extends the original EnOcean PTM 21xZ/ZE blueprint with continuous triggering capabilities. It allows you to perform repeated actions while a button is held down, perfect for scenarios like dimming lights gradually.

## Key Features

- **Initial Press**: Immediate action when button is first pressed
- **Hold Action**: Single action triggered after the hold delay
- **Continuous Actions**: Repeated actions while button remains held
- **Release Actions**: Action when button is released after being held

## Configuration

### Settings

- **Base topic**: MQTT base topic (default: `zigbee2mqtt`)
- **Hold delay**: Time before triggering hold actions (default: 500ms)
- **Repeat interval**: Time between continuous actions (default: 250ms)
- **PTM216Z**: Enable for PTM216Z-specific features

### Action Types

For each button combination, you can configure:

1. **Pressed**: Immediate action on button press
2. **Continuous**: Repeated action while held (after hold delay)
3. **Held**: Single action after hold delay
4. **Released**: Action when released after being held

## Usage Examples

### Example 1: Dimming Light with Button 1

```yaml
# Button 1 Pressed (immediate)
service: light.turn_on
target:
  entity_id: light.bedroom_light
data:
  brightness_pct: 100

# Button 1 Continuous (every 250ms while held)
service: light.turn_on
target:
  entity_id: light.bedroom_light
data:
  brightness_step_pct: -10  # Dim by 10% each time

# Button 1 Released
service: light.turn_off
target:
  entity_id: light.bedroom_light
```

### Example 2: Volume Control

```yaml
# Button 2 Continuous (increase volume)
service: media_player.volume_set
target:
  entity_id: media_player.living_room_speaker
data:
  volume_level: >
    {% set current = state_attr('media_player.living_room_speaker', 'volume_level') | float %}
    {{ [current + 0.05, 1.0] | min }}

# Button 3 Continuous (decrease volume)
service: media_player.volume_set
target:
  entity_id: media_player.living_room_speaker
data:
  volume_level: >
    {% set current = state_attr('media_player.living_room_speaker', 'volume_level') | float %}
    {{ [current - 0.05, 0.0] | max }}
```

## Technical Details

### Execution Flow

1. **Button Press Detected**:

   - Immediate execution of "Pressed" action
   - Start hold delay timer

2. **After Hold Delay**:

   - Execute "Held" action once
   - Start continuous loop with repeat interval

3. **Continuous Loop**:

   - Execute "Continuous" action
   - Wait for repeat interval
   - Repeat until button released

4. **Button Release**:
   - Stop continuous loop
   - Execute "Released" action if held longer than hold delay

### Mode Settings

- **Mode**: `restart` - New button press cancels previous automation
- **Max Exceeded**: `silent` - Prevents error messages when automation restarts

## Installation

1. Save the blueprint file to your Home Assistant blueprints directory:

   ```bash
   config/blueprints/automation/enocean_ptm21xz_continuous_trigger.yaml
   ```

2. Restart Home Assistant or reload automations

3. Create a new automation using this blueprint

4. Configure your button actions as needed

## Notes

- The continuous actions will repeat indefinitely while the button is held
- Use appropriate safeguards in your actions (e.g., min/max brightness limits)
- The automation uses `restart` mode, so pressing another button will cancel the current action
- For PTM216Z switches, make sure to create a Text Helper entity for state tracking

## Troubleshooting

- Ensure your EnOcean switch is properly paired with Zigbee2MQTT
- Check MQTT topic structure matches your Z2M configuration
- Verify the Text Helper entity exists for PTM216Z switches
- Monitor Home Assistant logs for any error messages during automation execution
