# Growatt Remote Power Control Implementation

## Overview

This implementation adds support for Growatt VPP (Virtual Power Plant) remote power control functionality as documented in the **Growatt VPP Communication Protocol of Inverter V2.01** (September 2024).

The remote power control feature allows direct control of battery charging and discharging behavior, enabling use cases such as:
- Dynamic energy arbitrage
- Grid services and demand response
- Peak shaving and load shifting
- Integration with external energy management systems

## Features Implemented

### 1. Remote Power Control Enable (Register 30407)
**Entity Type:** Select
**Entity ID:** `select.remote_power_control`
**Options:**
- `Disabled` (0) - Remote control is off
- `Enabled` (1) - Remote control is active

**Default:** Disabled
**Icon:** `mdi:remote`

### 2. Remote Charge/Discharge Power (Register 30409)
**Entity Type:** Number
**Entity ID:** `number.remote_charge_discharge_power`
**Range:** -100% to +100%
**Step:** 1%
**Unit:** Percentage

**Behavior:**
- **Positive values (1-100%):** Charge battery at X% of rated power
- **Zero (0%):** Do nothing / self-consumption mode
- **Negative values (-1 to -100%):** Discharge battery at X% of rated power

**Default:** 0%
**Icon:** `mdi:battery-sync`

### 3. Remote Control Duration (Register 30408)
**Entity Type:** Number
**Entity ID:** `number.remote_control_duration`
**Range:** 0 - 1440 minutes
**Step:** 1 minute
**Unit:** Minutes

**Behavior:**
- **0:** Unlimited duration (control remains active until changed)
- **1-1440:** Control is active for X minutes, then reverts to self-consumption

**Default:** 0 (unlimited)
**Icon:** `mdi:timer-outline`

### 4. Remote Control Actual Power (Register 30474)
**Entity Type:** Sensor (Read-only)
**Entity ID:** `sensor.remote_control_actual_power`
**Unit:** Percentage

**Purpose:** Provides feedback of the actual control value currently being executed by the inverter.

**Icon:** `mdi:battery-sync-outline`

## Usage Examples

### Example 1: Charge Battery at 50% Power for 2 Hours

```yaml
# Home Assistant Service Call
service: select.select_option
target:
  entity_id: select.remote_power_control
data:
  option: "Enabled"

service: number.set_value
target:
  entity_id: number.remote_charge_discharge_power
data:
  value: 50

service: number.set_value
target:
  entity_id: number.remote_control_duration
data:
  value: 120
```

### Example 2: Discharge Battery at 75% Power Indefinitely

```yaml
service: select.select_option
target:
  entity_id: select.remote_power_control
data:
  option: "Enabled"

service: number.set_value
target:
  entity_id: number.remote_charge_discharge_power
data:
  value: -75

service: number.set_value
target:
  entity_id: number.remote_control_duration
data:
  value: 0
```

### Example 3: Stop Remote Control (Return to Self-Consumption)

```yaml
service: select.select_option
target:
  entity_id: select.remote_power_control
data:
  option: "Disabled"
```

Or alternatively:

```yaml
service: number.set_value
target:
  entity_id: number.remote_charge_discharge_power
data:
  value: 0
```

## Control Logic Flow

The Growatt inverter follows this logic (as per protocol diagram on page 32):

```
1. Is Remote Power Control Enabled (register 30407)?
   ├─ NO → Use self-consumption or time-based charging schedule
   └─ YES → Continue to step 2

2. Check Remote Control Duration (register 30408)
   ├─ 0 → Unlimited time, continue to step 3
   └─ 1-1440 → Active for X minutes, then revert to self-consumption

3. Check Remote Charge/Discharge Power (register 30409)
   ├─ Positive (>0) → Charge battery at X% (Battery First priority)
   ├─ Zero (0) → Self-consumption mode (Load First priority)
   └─ Negative (<0) → Discharge battery at X% (Grid First priority)

4. Execute control and update Actual Power (register 30474)
```

## Compatibility

**Supported Inverters:**
- All Growatt generations: GEN, GEN2, GEN3, GEN4, SPF
- Includes: SPH, SPA, MIN, MIC, MOD, MID, WIT, WIS series

**Protocol Version:** VPP Communication Protocol V2.01 (2024-09-20)

## Safety Notes

1. **All entities are disabled by default** - You must manually enable them in Home Assistant
2. **Respects SOC limits** - The inverter will still honor configured SOC cutoff limits
3. **NOT storage** - Remote control registers (30407, 30408, 30409) are marked "Not storage" in the protocol, meaning values are not persisted across inverter reboots
4. **Priority interaction** - Remote control overrides time-based charging schedules when enabled

## Testing

Before deploying to production:

1. **Enable the entities** in Home Assistant (they are disabled by default)
2. **Test with small values** first (e.g., ±10%)
3. **Monitor the Actual Power sensor** (register 30474) to verify commands are executing
4. **Check battery SOC limits** are still being respected
5. **Test duration timeout** to ensure control reverts as expected

## References

- **Protocol Document:** `docs/growatt/2.1 GROWATT VPP COMMUNICATION PROTOCOL OF INVERTER_V2.01.pdf`
- **Key Sections:**
  - Page 13-14: Battery Power Control Set Information (registers 30300-30499)
  - Page 32: Remote power control schematic diagram
  - Section 3.5: Remote power control logic flow

## Implementation Details

**Files Modified:**
- `custom_components/solax_modbus/plugin_growatt.py`

**Registers Mapped:**
- 30407 → Register 407 (Remote power control enable)
- 30408 → Register 408 (Remote control duration)
- 30409 → Register 409 (Remote charge/discharge power)
- 30474 → Register 474 (Actual control value - read-only)

**Entity Configuration:**
- All entities use `allowedtypes = ALL_GEN_GROUP` for maximum compatibility
- All entities are `entity_registry_enabled_default = False` for safety
- Write operations use `WRITE_SINGLE_MODBUS` method
- Sensor uses `REG_HOLDING` register type (as per protocol spec)
