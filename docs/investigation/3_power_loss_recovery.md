> **Status:** Investigation notes for [iot-root/garden-of-eden#3](https://github.com/iot-root/garden-of-eden/issues/3) ("power loss recovery"). Captures suspected root causes and proposed fixes prior to implementation.
> **Delete when:** issue #3 is closed/fixed. If fixes land, distill any durable operational notes into `README.md` (or a new ops doc) before deleting.
> **Conclusions distilled to:** not yet — pending fix PR.

# Power Loss Recovery (Issue #3)

> *"Currently system has to be cycled a few times after a power loss."*

## Most likely root cause: `mqtt.service` exhausts systemd's restart burst before the broker is ready

The unit template in `bin/setup.sh:238-252` only declares:

```ini
Requires=pigpiod.service
After=network.target pigpiod.service
```

There is **no** `After=mosquitto.service` or `Wants=mosquitto.service`. So on a cold power-up, systemd starts `mqtt.service` and `mosquitto` in parallel.

`mqtt.py:528` then does:

```python
client.connect(BROKER, PORT, KEEP_ALIVE_INTERVAL)
```

This is the **synchronous** `connect()` — if mosquitto isn't accepting connections yet, it raises `ConnectionRefusedError` and the Python process dies. The `loop_forever()`-driven reconnect logic from paho-mqtt only kicks in *after* a successful initial connection, so it doesn't help here.

`Restart=always` is set, but with no `RestartSec=` (defaults to 100ms) and no `StartLimitBurst=` override. systemd's defaults are **5 restarts within 10 seconds, then give up**. That matches the symptom exactly: the first boot burns through 5 instant-restart attempts in ~half a second while mosquitto is still starting, systemd marks the unit failed, and the user has to power-cycle (or run `systemctl reset-failed && systemctl start mqtt.service`) until they happen to hit a boot where mosquitto comes up fast enough.

## Secondary suspects

- **pigpiod socket race.** `Requires=pigpiod.service` makes systemd start pigpiod, but only waits for the unit to be marked active — not for the pigpio TCP socket to be listening. `mqtt.py:44-48` instantiates `PiGPIOFactory()` and `Pump`/`Light`/`Distance` at module import time. If pigpio's socket isn't up yet, this raises before `client.connect` is even reached. Would surface as `gpiozero.exc.BadPinFactory` or a pigpio connection error in `gardyn.log` on the first boot after power loss.
- **AM2320 wakeup quirk** (Gardyn 1.0/2.0 only). `bin/setup.sh:172-184` notes AM2320 doesn't appear on i2cdetect without a wakeup sequence. Cold-boot sensor init could fail on these older models.

## Suggested fixes (in order of payoff)

### 1. Add broker dep + restart pacing to the systemd unit

In `bin/setup.sh:setup_mqtt_service`:

```ini
[Unit]
Requires=pigpiod.service
Wants=mosquitto.service
After=network-online.target pigpiod.service mosquitto.service

[Service]
Restart=always
RestartSec=10
StartLimitBurst=0
```

Highest-leverage change. Requires either re-running `setup.sh` or manually editing `/etc/systemd/system/mqtt.service` on existing installs.

### 2. Switch initial connect to async + retry-aware loop

In `mqtt.py:528`:

```python
client.connect_async(BROKER, PORT, KEEP_ALIVE_INTERVAL)
...
client.loop_forever(retry_first_connection=True)
```

Makes the process resilient even if mosquitto hiccups mid-life, not just at boot.

### 3. (Optional) Retry module-level sensor init

Only worth doing if logs confirm a pigpiod socket race. Wrap the `Pump(...)`, `Light(...)`, `Distance(...)` calls at `mqtt.py:46-48` in a small bounded retry.

## Recommendation

Bundle #1 and #2 in one PR. Hold #3 unless `gardyn.log` shows GPIO/pigpio errors on first boot after power loss.
