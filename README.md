# pyIntesisHome

## Experimental branch: `poc_timeouts`

> **Note:** This is a personal fork of [jnimmo/pyIntesisHome](https://github.com/jnimmo/pyIntesisHome). The `poc_timeouts` branch contains experimental fixes for regular cloud connection timeouts that affect IntesisHome, anywAir, and airconwithme devices.

### Problem

Cloud-connected devices use a persistent TCP connection that regularly drops silently due to NAT/firewall idle timeouts (typically 30–90 seconds). The upstream library only sends a keepalive every 120 seconds, so a dead connection can go undetected for up to 2–10 minutes. During this window, Home Assistant sees the device as connected but commands are lost and state updates stop arriving.

### What this branch experiments with

1. **Reduced keepalive interval** — from 120 s down to 30 s, keeping the connection alive through most NAT devices
2. **Read timeout on `readuntil`** — wraps the blocking read in `asyncio.wait_for(..., timeout=180)` so zombie half-open TCP connections are detected promptly
3. **OS-level TCP keepalive** — enables `SO_KEEPALIVE` on the socket so the kernel probes the connection independently of the application layer
4. **Cleaner auth wait** — replaces a 100 ms busy-poll loop with `asyncio.wait_for` on the response event

This branch is used by the [engelchrisi/hass-intesishome](https://github.com/engelchrisi/hass-intesishome) fork for end-to-end testing in a real Home Assistant environment.

---

This project is a python3 library for interfacing with Intesis air conditioning controllers, including cloud control of IntesisHome (Airconwithme + anywAiR) and local control of IntesisBox devices.
It is fully asynchronous using the aiohttp library, and utilises the private API used by the IntesisHome mobile apps.

### Home Assistant

To use with [Home Assistant](https://www.home-assistant.io/integrations/intesishome/), add the following to your configuration.yaml

#### IntesisHome configuration example

```yaml
climate:
  - platform: intesishome
    username: YOUR_USERNAME
    password: YOUR_PASSWORD
```

#### IntesisBox configuration example

```yaml
climate:
  - platform: intesishome
    device: IntesisBox
    host: 192.168.1.50
```

## Library usage

- Instantiate the IntesisHome controller device with username and password for the user.intesishome.com website.
- Status can be polled using the poll_status command suggested maximum of once every 5 minutes.
- Commands are sent using a TCP connection to the API which will then remain open until the connection times out.
- While the persistent TCP connection is open, status updates are pushed to the device over the socket meaning polling is not required (check using _is_connected_ property)
- Callbacks to be notified of state updates can be added with the add_callback() method.

### Library basic example

```python
import asyncio
from pyintesishome import IntesisHome

async def main(loop):
    controller = IntesisHome('username', 'password', loop=loop, device_type='airconwithme')
    await controller.connect()
    print(repr(controller.get_devices()))
    # Imagine you have a device with id 12015601252591
    if await controller.get_power_state('12015601252591') == 'off':
        await controller.set_power_on('12015601252591')

    await controller.set_mode_heat('12015601252591')
    await controller.set_temperature('12015601252591', 22)
    await controller.set_fan_speed('12015601252591','quiet')

if __name__ == "__main__":
    loop = asyncio.get_event_loop()
    result = loop.run_until_complete(main(loop))

```

### Control methods

- set_mode_heat(deviceID)
- set_mode_cool(deviceID)
- set_mode_fan(deviceID)
- set_mode_dry(deviceID)
- set_mode_auto(deviceID)
- set_temperature(deviceID, temperature)
- set_fan_speed(deviceID, 'quiet' | 'low' | 'medium' | 'high' | 'auto')
- set_power_on(deviceID)
- set_power_off(deviceID)
