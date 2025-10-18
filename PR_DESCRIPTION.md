# Add Growatt VPP (Virtual Power Plant) Remote Power Control Support

## Summary

This PR adds support for Growatt VPP remote power control functionality, allowing Home Assistant to remotely control battery charging/discharging based on the Growatt VPP Communication Protocol V2.01 (September 2024).

## Problem Statement

Growatt inverters with firmware ZBDC-0014 or newer support VPP remote power control through modbus registers. This functionality is documented in the official Growatt VPP Communication Protocol but is not currently exposed in the Home Assistant integration.

**Modpoll testing confirms the registers work correctly:**
- Register 30100 (Control Authority) can be written and read back successfully
- Registers 30407-30410 can be written as a group using FC16 (write multiple registers)
- Register 30474 (Actual Power) correctly reflects the commanded power value
- VPP control successfully commands the battery to charge/discharge at the specified power level

## Current Issue - Need Help!

We've implemented VPP support following existing patterns in the integration (similar to Time-based controls and inverter_switch), but **the entities show "unknown" and writes from Home Assistant don't reach the inverter**, despite modpoll working perfectly.

**What works via modpoll:**
- ✅ Write to register 30101 (Control Authority) using FC16 - value persists
- ✅ Write multiple to registers 30408-30411 - all values write correctly
- ✅ Register 30475 correctly shows actual power after VPP commands
- ✅ Battery responds to VPP commands and charges/discharges as commanded

**What doesn't work via Home Assistant:**
- ❌ All VPP select/number entities show "unknown"
- ❌ Writes from Home Assistant entities don't reach the inverter
- ❌ Register reads don't populate entity values (despite logs showing "treating register 0x64")

## Implementation Approaches Tried

### Approach 1: Direct Write Pattern (following `inverter_switch`)
- Select entity with `WRITE_MULTISINGLE_MODBUS` and `register = 100`
- Internal sensor with `scale = {0: "Disabled", 1: "Enabled"}`
- Result: Entity shows "unknown", writes don't work

### Approach 2: Local + Button Pattern (following Time controls)
- Select/number entities with `WRITE_DATA_LOCAL`
- Button entities with `WRITE_MULTI_MODBUS` and value_functions
- Internal `register_XXX` sensors for value_function access
- Read-only sensors with value_functions for display
- Result: Entities still show "unknown"

## Code Changes

### 1. Value Functions (lines 540-571)
```python
def value_function_vpp_control_update(initval, descr, datadict):
    """Write VPP remote control registers 407-410"""
    remote_power_control = datadict.get('vpp_remote_power_control', 'Disabled')
    remote_duration = datadict.get('vpp_remote_control_duration', 0)
    remote_power = datadict.get('vpp_remote_charge_discharge_power', 0)

    enable_value = 1 if remote_power_control == 'Enabled' else 0

    return [
        (REGISTER_U16, enable_value),           # Register 407
        (REGISTER_U16, int(remote_duration)),   # Register 408
        (REGISTER_S16, int(remote_power)),      # Register 409
        (REGISTER_U16, 1),                      # Register 410
    ]

def value_function_vpp_control_clear(initval, descr, datadict):
    """Clear VPP remote control registers"""
    return [(REGISTER_U16, 0), (REGISTER_U16, 0), (REGISTER_S16, 0), (REGISTER_U16, 0)]

def value_function_vpp_remote_power_control_read(initval, descr, datadict):
    """Read enable status from register 407"""
    value = datadict.get('register_407', 0)
    return "Enabled" if int(value) == 1 else "Disabled"
```

### 2. Entities Added

**Control Entities (local + button commit):**
- `select.vpp_remote_power_control` - Enable/Disable (WRITE_DATA_LOCAL)
- `number.vpp_remote_control_duration` - Duration 0-1440 minutes (WRITE_DATA_LOCAL)
- `number.vpp_remote_charge_discharge_power` - Power -100% to +100% (WRITE_DATA_LOCAL)
- `button.vpp_control_update` - Commits to registers 407-410 (WRITE_MULTI_MODBUS)
- `button.vpp_control_clear` - Resets all to 0 (WRITE_MULTI_MODBUS)

**Authority Control (direct write):**
- `select.vpp_control_authority` - Register 100 (WRITE_MULTISINGLE_MODBUS)

**Read-Only Status:**
- `sensor.vpp_remote_power_control_read` - Shows actual value from register 407
- `sensor.vpp_remote_control_duration_read` - Shows actual value from register 408
- `sensor.vpp_remote_charge_discharge_power_read` - Shows actual value from register 409
- `sensor.vpp_remote_control_actual_power` - Shows actual power from register 474

**Internal Sensors:**
- Internal sensor for `vpp_control_authority` with scale dict
- Internal sensors `register_407`, `register_408`, `register_409` for value_function access

## Questions for Maintainers

1. **Why do entities show "unknown"?**
   - Logs show registers being read: `treating register 0x64 : vpp_control_authority`
   - Internal sensor has correct `scale` dict to convert 0/1 to "Disabled"/"Enabled"
   - What are we missing?

2. **Why don't writes work?**
   - `WRITE_MULTISINGLE_MODBUS` for register 100 - logs show "writing...register 100 value 1 with method 2"
   - But modpoll shows register stays at 0
   - Same write via modpoll works perfectly

3. **Is there something special about these register addresses?**
   - Register 100 is in a different range than typical Growatt registers
   - Could there be a block size or range limitation?

4. **Pattern validation:**
   - Are we following the correct pattern for WRITE_DATA_LOCAL + button entities?
   - Is the value_function return format correct for multi-register writes?
   - Should internal sensors be set up differently?

## Register Mapping

| PDF Doc | HA Code | modpoll | Description | Type | Range |
|---------|---------|---------|-------------|------|-------|
| 30100 | 100 | 30101 | Control Authority | U16 | 0=Disabled, 1=Enabled |
| 30407 | 407 | 30408 | Remote Power Enable | U16 | 0=Disabled, 1=Enabled |
| 30408 | 408 | 30409 | Duration | U16 | 0-1440 minutes |
| 30409 | 409 | 30410 | Charge/Discharge Power | S16 | -100 to +100 % |
| 30410 | 410 | 30411 | Unknown/Reserved | U16 | Always 1 in tests |
| 30474 | 474 | 30475 | Actual Power (read) | S16 | Current power % |

Note: modpoll uses 1-based addressing, so PDF register + 1

## Successful modpoll Test Sequence

```bash
# Enable Control Authority (register 30100)
$ ./modpoll -m tcp -a 1 -r 30101 -t 4 -1 192.168.0.52 1
Protocol configuration: MODBUS/TCP, FC16
Written 1 reference.

# Verify
$ ./modpoll -m tcp -a 1 -r 30101 -t 4 -1 192.168.0.52
[30101]: 1  ✅

# Write VPP control (enable=1, duration=2min, power=-8%, unknown=1)
$ ./modpoll -m tcp -a 1 -r 30408 -c 4 -t 4 -1 192.168.0.52 1 2 -8 1
Protocol configuration: MODBUS/TCP, FC16
Written 4 references.

# Check actual power
$ ./modpoll -m tcp -a 1 -r 30475 -t 4 -1 192.168.0.52
[30475]: -8  ✅ Battery discharging at 8%!
```

## Test Environment

- **Inverter:** Growatt DN1.0 (GEN4 | HYBRID | X3)
- **Firmware:** ZBDC-0014 (VPP support confirmed by Growatt support)
- **HA Integration:** homeassistant-solax-modbus (latest)
- **Modbus:** TCP/IP on 192.168.0.52:502, slave ID 1

## Files Modified

- `custom_components/solax_modbus/plugin_growatt.py`

## Request for Help

We've tried following existing patterns but are clearly missing something about how the integration handles:
- Entity value population from register reads
- Write operations reaching the inverter
- Internal sensor to entity value propagation

Any guidance would be greatly appreciated! The modbus communication works perfectly via modpoll, so we know the inverter supports this, we just can't get the HA integration to properly read/write these registers.

## Supporting Materials

- Growatt VPP Communication Protocol V2.01 PDF (will attach screenshots)
- modpoll test output showing successful operations
- Home Assistant debug logs showing register reads but "unknown" entities
