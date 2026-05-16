# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Garden of Eden is a Raspberry Pi-based controller for hydroponic gardens (a DIY replacement/companion for the Gardyn). It runs on the Pi itself and talks to physical hardware (pumps, lights, sensors, camera, momentary button) over GPIO/I2C.

Hardware access is required to actually drive sensors, but the unit tests under `tests/` mock GPIO and I2C heavily (`@patch` on `PWMLED`, `PiGPIOFactory`, route-level driver instances), so `python -m unittest` runs fine on a dev machine without a Pi.

## Two-process architecture

There are two independent long-running processes that share the same sensor driver code under `app/sensors/`:

1. **Flask REST API** (`run.py` → `app/__init__.py`) — HTTP on port 5000. Each sensor registers a blueprint (e.g. `/light`, `/pump`, `/distance`). Used for ad-hoc control and `bin/api-test.sh`.
2. **MQTT bridge** (`mqtt.py`, run as `mqtt.service`) — connects to a Mosquitto broker for HomeAssistant integration, publishes sensor readings on a schedule, subscribes to command topics, and owns the **momentary button** handler.

Both processes import the same sensor classes from `app/sensors/<name>/<name>.py`. The third entry point is the CLI: each driver module is also runnable standalone (`python app/sensors/pump/pump.py --on --speed 100`), which is how cron-driven scheduling works.

**GPIO contention gotcha:** drivers are instantiated at *module import* time — see `app/sensors/pump/routes.py:7` (`pump_control = PumpControl()`) and `mqtt.py:46` (`pump = Pump(...)`). On the same Pi you cannot run `mqtt.service` and `run.py` concurrently against real hardware — both would try to claim the same pins. Stop the service before running the Flask app for hands-on testing.

The system also depends on `pigpiod` (a separate system daemon). `gpiozero` is configured with `PiGPIOFactory` so GPIO is hardware-timed rather than CPU-polled — this matters for PWM accuracy and the ultrasonic distance sensor.

## Sensor module convention

Every sensor under `app/sensors/<name>/` follows the same three-file pattern:

- `<name>.py` — driver class + `argparse` CLI entry point
- `routes.py` — Flask blueprint exposing the driver over HTTP
- `__init__.py`

When adding a new sensor, mirror this layout and register the blueprint in `app/__init__.py:create_app`. If it should also be exposed over MQTT or driven by the button, wire it up in `mqtt.py`.

## Momentary button behavior

Defined in `mqtt.py` (GPIO 13, `bounce_time=0.2`, `hold_time=2`):

- **Single press** → toggle light
- **Double press** (within 1s) → toggle pump
- Long-press (`hold_time=2`) is configured but currently has no action wired up

Press-timing constants live at the top of `mqtt.py`.

## Common commands

```bash
# one-time setup on a fresh Pi (installs OS deps, python libs, starts pigpiod + mqtt.service)
./bin/setup.sh

# Flask REST API (foreground, port 5000)
source venv/bin/activate && python run.py

# tests
python -m unittest -v              # all
python tests/test_distance.py      # single file

# REST endpoint smoke test
./bin/api-test.sh

# control a single sensor from CLI (same entry points cron uses)
python app/sensors/light/light.py --on --brightness 50
python app/sensors/pump/pump.py --on --speed 100

# system service status
sudo systemctl status pigpiod mqtt.service mosquitto

# tail MQTT bridge logs
./bin/show-mqtt-logs.sh
```

`requirements.txt` pins both `dotenv==0.0.5` (a stub package) and `python-dotenv==1.0.1`; they ship the same top-level `dotenv` module and conflict. If `run.py` errors with `AttributeError: module 'dotenv' has no attribute 'find_dotenv'`, see the troubleshooting note in `README.md` under REST API.

`bin/setup.sh` also installs `/usr/local/bin/light` and `/usr/local/bin/water` symlinks (pointing at `bin/light.sh` / `bin/water.sh`) — that's where those bare command names come from on a provisioned Pi.

## Configuration

All runtime config flows through `config.py` from `.env` (copy `.env-dist`). MQTT broker, camera devices, image paths, water-low threshold, etc. are all env-driven — don't hardcode them.

`SENSOR_TYPE` is special: it's appended to `.env` by `bin/setup.sh` based on what `i2cdetect` finds (`DHT20` for Gardyn 3.0+, `AM2320` for 1.0/2.0). The temperature/humidity drivers branch on this value, so changing it switches which CircuitPython library is used.

## Commit style

Conventional commits are required — see `CONTRIBUTORS.md`. Format: `<type>(<scope>): <description>` where type is one of `feat|fix|docs|style|refactor|test|chore`. Breaking changes use `!` after the scope and a `BREAKING CHANGE:` footer.

## Hardware reference

Pin assignments, I2C addresses, and sensor part numbers are in `README.md` under "Hardware Overview". Run `sudo i2cdetect -y 1` on the Pi to confirm sensors are visible at their expected addresses (0x38 AM2320, 0x40 INA219, 0x48 PCT2075).
