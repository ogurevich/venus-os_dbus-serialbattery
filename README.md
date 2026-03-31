# dbus-serialbattery (fork: ogurevich)

This is a personal fork of [mr-manuel/venus-os_dbus-serialbattery](https://github.com/mr-manuel/venus-os_dbus-serialbattery), adding the `UTILIZE_SOC_OF_DBUS_SERVICE` feature on branch `ogdev`.

For general documentation, supported BMS list, installation instructions and troubleshooting please refer to the **upstream project**:

* [Introduction](https://mr-manuel.github.io/venus-os_dbus-serialbattery_docs/)
* [Features](https://mr-manuel.github.io/venus-os_dbus-serialbattery_docs/general/features)
* [Supported BMS](https://mr-manuel.github.io/venus-os_dbus-serialbattery_docs/general/supported-bms)
* [How to install, update, disable, enable and uninstall](https://mr-manuel.github.io/venus-os_dbus-serialbattery_docs/general/install)
* [How to troubleshoot](https://mr-manuel.github.io/venus-os_dbus-serialbattery_docs/troubleshoot/)
* [FAQ](https://mr-manuel.github.io/venus-os_dbus-serialbattery_docs/faq/)

## What this fork adds: `UTILIZE_SOC_OF_DBUS_SERVICE`

### Problem

BMS SoC values drift over time. After charge/discharge cycles the internal BMS coulomb counter diverges from reality. A SmartShunt measuring at the DC busbar has a more accurate, system-wide SoC.

Without this feature the driver stays in **Bulk mode** long after the batteries are effectively full, because `SWITCH_TO_BULK_SOC_THRESHOLD` is never reached by the drifted BMS SoC.

### Why not `EXTERNAL_SENSOR_DBUS_PATH_SOC`?

The upstream setting `EXTERNAL_SENSOR_DBUS_PATH_SOC` replaces the BMS SoC **everywhere on D-Bus**, including the published `/Soc` path. This breaks **multi-battery + aggregator setups**:

- A SmartShunt measures total system current → its SoC represents the combined system.
- Each battery driver publishes to its own D-Bus service (`com.victronenergy.battery.ttyUSB0`, etc.).
- The battery aggregator reads individual `/Soc` values to compute a weighted average.
- If all drivers replace their `/Soc` with the same SmartShunt value, the aggregator receives N identical values and the weighted average becomes meaningless.

### Solution

`UTILIZE_SOC_OF_DBUS_SERVICE` uses the external SoC **only for the internal CVL Float/Bulk switching decision**, while the original BMS SoC is still published unchanged on D-Bus `/Soc`.

```
SmartShunt /Soc  ──►  get_utilized_soc()  ──►  manage_charge_voltage()
                                                (Bulk/Float decision only)

BMS /Soc  ──────────────────────────────────►  D-Bus /Soc  (unchanged)
                                               (aggregator sees real BMS SoC)
```

### Configuration

```ini
; In config.ini, set the D-Bus service name of the external SoC source.
; Example: SmartShunt or battery aggregator service.
; Do NOT combine with EXTERNAL_SENSOR_DBUS_PATH_SOC.
UTILIZE_SOC_OF_DBUS_SERVICE = com.victronenergy.battery.ttyS5
```

## Developer Notes

This project makes use of `velib_python`, pre-installed on Venus OS under `/opt/victronenergy/dbus-systemcalc-py/ext/velib_python`. To use the Python files locally, `git clone` [velib_python](https://github.com/victronenergy/velib_python) and add it to `PYTHONPATH`.

#### How it works

* Each supported BMS implements the abstract base class `Battery` from `battery.py`.
* `dbus-serialbattery.py` detects the connected BMS by calling `test_connection()` on each known implementation. On success it periodically calls `dbushelper.publish_battery()`, which runs `Battery.refresh_data()` and publishes the updated fields to dbus via `dbushelper.publish_dbus()`.
* The Victron device is controlled by values published on `/Info/`:
  * `/Info/MaxChargeCurrent`
  * `/Info/MaxDischargeCurrent`
  * `/Info/MaxChargeVoltage`
  * `/Info/BatteryLowVoltage`

For more details see the [official Victron dbus documentation](https://github.com/victronenergy/venus/wiki/dbus).

## Credits

Original project by [Louisvdw](https://github.com/Louisvdw/dbus-serialbattery).
Maintained since 2023 by [mr-manuel](https://github.com/mr-manuel).
This fork by [ogurevich](https://github.com/ogurevich).

## Help translating to your language

Are you using this driver and you would like to have it in your language? Now you can help to translate it on [POEditor](https://poeditor.com/join/project/sA2DhyEpYh).
