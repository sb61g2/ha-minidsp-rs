# Contributing / Development notes

## Repo relationship

This repo (`sb61g2/ha-minidsp-rs`) is the **HA distribution package** — it is what users add to HACS and the add-on store. It contains:

- `custom_components/minidsp/` — the HA custom integration
- `addon/` — the HAOS add-on config (references pre-built images at `ghcr.io/sb61g2/minidsp-rs-addon-{arch}`)
- `blueprints/` — the watchdog automation blueprint

The integration source also lives in the **daemon repo** (`sb61g2/minidsp-rs`) at `homeassistant-integration/custom_components/minidsp/`. The two copies must be kept in sync manually — there is no automated mirroring.

### Syncing changes

When you edit the integration in **either** repo, mirror the change to the other:

```bash
# After editing in ha-minidsp-rs, copy to minidsp-rs:
cp custom_components/minidsp/<file> \
   ../minidsp-rs/homeassistant-integration/custom_components/minidsp/<file>

# After editing in minidsp-rs, copy to ha-minidsp-rs:
cp ../minidsp-rs/homeassistant-integration/custom_components/minidsp/<file> \
   custom_components/minidsp/<file>
```

Commit and push both repos. The `minidsp-rs` integration changes go on the `dev` branch.

---

## WebSocket design in coordinator.py

### Why `async for msg in ws:` instead of `asyncio.wait_for`

The coordinator uses:

```python
async with self._session.ws_connect(url, heartbeat=60) as ws:
    self._ws_connected[device_index] = True
    async for msg in ws:
        ...
```

An earlier version used `asyncio.wait_for(ws.receive(), timeout=120)` to detect hung connections. This caused **entity-unavailability blips every 2 minutes** during idle (when the MiniDSP daemon sends no unsolicited messages because nothing has changed).

The attempted fix — adding `heartbeat=60` to `ws_connect` — did not resolve it. aiohttp's WebSocket heartbeat sends PING frames and consumes the PONG at the transport layer. The PONG **never appears in `ws.receive()`**, so the 120-second `wait_for` timeout continued to fire on schedule regardless.

`async for msg in ws:` is the correct approach: it blocks until the server closes the connection or an error occurs, and exits naturally. The `heartbeat=60` parameter is still passed to keep the underlying TCP connection alive through NAT/firewalls, but availability is tracked by WS connection state, not message recency.

### Availability tracking

`is_device_available()` returns `self._ws_connected.get(device_index, False)` — a bool that is set `True` when the WS connects and `False` in the `finally` block when it closes or errors. This immediately reflects connection state to HA entities without relying on message timestamps.

Listeners are fired on both connect and disconnect, so HA entities update availability the moment the connection changes — no polling lag.
