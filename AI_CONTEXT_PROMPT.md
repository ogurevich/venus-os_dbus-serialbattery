# AI Context: UTILIZE_SOC_OF_DBUS_SERVICE Patch

## What this branch adds

Branch `ogdev` adds the feature `UTILIZE_SOC_OF_DBUS_SERVICE` on top of the upstream
`mr-manuel/venus-os_dbus-serialbattery` driver. It was ported from a 2+ year production
fork of `Louisvdw/dbus-serialbattery` (see [discussion #727](https://github.com/Louisvdw/dbus-serialbattery/discussions/727)).

## The Problem

BMS SoC values drift over time. After charge/discharge cycles, the internal BMS coulomb
counter diverges from reality. A SmartShunt measuring at the DC busbar has a more
accurate, system-wide SoC.

The consequence without this patch: the driver stays in **Bulk mode** (applying full
charge voltage) long after the batteries are effectively full, because
`SWITCH_TO_BULK_SOC_THRESHOLD` is never reached by the drifted BMS SoC.

## Why Not Use the Existing `EXTERNAL_SENSOR_DBUS_PATH_SOC`?

The upstream setting `EXTERNAL_SENSOR_DBUS_PATH_SOC` solves a similar problem but with
a fundamentally different and wider scope: it **replaces the BMS SoC everywhere on D-Bus**,
including on the published `/Soc` path. The original BMS SoC is moved to `/SocBms`.

This is **wrong for multi-battery + aggregator setups**:

- A SmartShunt measures **total system current** → its SoC represents the combined system.
- In a setup with N batteries in parallel, each battery driver publishes to its own
  D-Bus service (e.g. `com.victronenergy.battery.ttyUSB0`, `...ttyUSB1`, etc.).
- The battery aggregator driver reads the individual `/Soc` values from each service
  to compute a weighted average.
- If all drivers replace their `/Soc` with the same SmartShunt value, **every driver
  publishes the same number**. The aggregator receives N identical values and the
  weighted average becomes meaningless.

The aggregator must receive **individual BMS SoC values** to function correctly.

## What This Patch Does Instead

`UTILIZE_SOC_OF_DBUS_SERVICE` uses the external SoC value **only for the internal
CVL Float/Bulk switching decision** (`SWITCH_TO_BULK_SOC_THRESHOLD`), while the
original BMS SoC is still published unchanged on D-Bus `/Soc`.

```
SmartShunt /Soc  ──►  get_utilized_soc()  ──►  manage_charge_voltage_limit()
                                                (Bulk/Float decision only)

BMS /Soc  ──────────────────────────────────►  D-Bus /Soc  (unchanged)
                                               (aggregator sees real BMS SoC)
```

## Configuration

```ini
; In config.ini, set the D-Bus service name of the external SoC source.
; Example: SmartShunt or battery aggregator service.
; Do NOT combine with EXTERNAL_SENSOR_DBUS_PATH_SOC.
UTILIZE_SOC_OF_DBUS_SERVICE = com.victronenergy.battery.ttyS5
```

## Files Changed

| File | Change |
|---|---|
| `dbus-serialbattery/config.default.ini` | New config section with documentation |
| `dbus-serialbattery/utils.py` | Load `UTILIZE_SOC_OF_DBUS_SERVICE` from config |
| `dbus-serialbattery/battery.py` | `get_system_dc_battery_soc` field in `init_values()`; new method `get_utilized_soc()`; two usages of `soc_calc` in `manage_charge_voltage_limit()` replaced with `get_utilized_soc()` |
| `dbus-serialbattery/dbushelper.py` | `_dbusBatterySocItem` field in `__init__`; `VeDbusItemImport` setup at end of `setup_vedbus()`; new method `get_dbus_battery_soc()` |

## Key Implementation Details

### `battery.py` — `get_utilized_soc()`

```python
def get_utilized_soc(self) -> float:
    if self.get_system_dc_battery_soc:
        try:
            systemDcBatterySoc = self.get_system_dc_battery_soc()
            if systemDcBatterySoc is not None:
                return systemDcBatterySoc
        except Exception:
            pass
    return self.soc_calc if self.soc_calc is not None else self.soc
```

- Falls back to `soc_calc` (not `self.soc`) to stay consistent with the upstream CVL logic which uses `soc_calc` as its canonical SoC value.
- Exception-safe: any D-Bus read error is silently absorbed and the BMS value is used.

### `dbushelper.py` — Setup in `setup_vedbus()`

The `VeDbusItemImport` is created at the end of `setup_vedbus()`, after `self._dbusservice.register()`, because the D-Bus main loop is available at that point. The assignment `self.battery.get_system_dc_battery_soc = lambda: self.get_dbus_battery_soc()` injects the reader into the battery object without changing the `Battery` class interface.

## Hardware Setup This Was Tested On

- Cerbo GX running Venus OS
- 3× LiFePO4 battery packs in parallel
- `dbus-serialbattery` driver instance per pack (one per serial port)
- Battery aggregator driver combining all packs
- Victron SmartShunt as external SoC/current source

## Production History

Running continuously since July 2023 without issues. Fallback to `soc_calc` activates
correctly during SmartShunt unavailability (e.g. reboot).

## Related Links

- Original discussion: https://github.com/Louisvdw/dbus-serialbattery/discussions/727
- Source fork (old base): https://github.com/ogurevich/dbus-serialbattery/tree/ogdev
- This fork: https://github.com/ogurevich/venus-os_dbus-serialbattery/tree/ogdev
- Upstream `EXTERNAL_SENSOR_DBUS_PATH_SOC` (alternative, not suitable for multi-battery): see `config.default.ini` section "External Sensor for Current and/or SoC"
