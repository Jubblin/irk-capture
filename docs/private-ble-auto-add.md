# Auto-Add Captured IRKs to Private BLE Device

This optional add-on watches the IRK Capture **IRK** sensor and automatically
creates a Home Assistant **Private BLE Device** config entry when a new key
is captured, instead of you copy/pasting the IRK into the UI by hand.

## How it works, and its one hard limitation

`private_ble_device` is a config-flow-only integration: there is no YAML or
service-call way to add an entry. Its setup flow takes a single field (the
IRK) and, before it will let you finish, checks that **Home Assistant's own
Bluetooth scanner** has already seen and resolved a live advertisement for
that IRK.

That scanner is *not* the IRK Capture ESP32 — this package is intentionally
single-purpose and never acts as a Bluetooth proxy (see the main
[README](../README.md)). So the phone/watch has to be seen by whatever
Bluetooth adapter or ESPHome proxy Home Assistant normally uses for BLE, and
that can lag a few seconds to a couple of minutes behind the capture. This
automation handles that by retrying the submission on a delay instead of
giving up after one try.

Because this relies on Home Assistant's internal `config_entries/flow` REST
API (the same one the frontend uses, not a documented public API), it can in
principle change between Core releases. If it stops working after an
upgrade, check the persistent notifications this automation creates for the
error, and fall back to adding the device manually.

## Setup

### 1. Create a Long-Lived Access Token

In Home Assistant: your profile (bottom left) → **Security** tab → **Long-Lived
Access Tokens** → **Create Token**. Copy it immediately; it's shown once.
This token needs to belong to an **admin** account — creating a config entry
requires admin privileges.

### 2. Add the token to `secrets.yaml`

```yaml
irk_capture_ha_token: "Bearer PASTE_YOUR_TOKEN_HERE"
```

Keep the `Bearer` prefix (with the trailing space before the token) — it's
used directly as the header value.

### 3. Add the two `rest_command` entries to `configuration.yaml`

```yaml
rest_command:
  irk_capture_start_ble_flow:
    url: "http://localhost:8123/api/config/config_entries/flow"
    method: POST
    headers:
      Authorization: !secret irk_capture_ha_token
      Content-Type: application/json
    payload: '{"handler": "private_ble_device", "show_advanced_options": false}'
    timeout: 10

  irk_capture_submit_irk:
    url: "http://localhost:8123/api/config/config_entries/flow/{{ flow_id }}"
    method: POST
    headers:
      Authorization: !secret irk_capture_ha_token
      Content-Type: application/json
    payload: '{"irk": "{{ irk }}"}'
    timeout: 10
```

If Home Assistant isn't reachable at `localhost:8123` from itself (unusual,
but possible with some reverse-proxy/container setups), change the `url`
host/port accordingly.

Restart Home Assistant (or use **Developer Tools → YAML → Reload rest_command
entities** if just reloading config) to pick these up.

### 4. Import the blueprint

Settings → Automations & Scenes → **Blueprints** tab → **Import Blueprint**,
and paste:

```text
https://raw.githubusercontent.com/DerekSeaman/irk-capture/main/blueprints/automation/DerekSeaman/irk_auto_add_private_ble.yaml
```

### 5. Create the automation

From the imported blueprint, click **Create Automation**, and set:

- **IRK Sensor** — your IRK Capture device's `IRK` text sensor entity.
- **Max Retry Attempts** / **Retry Delay** — defaults (12 attempts, 15s
  apart, so ~3 minutes total) are reasonable; increase if your devices are
  slow to re-advertise.

### 6. Test it

Pair a device to the ESP32 as usual. Once the IRK sensor updates, watch for
a persistent notification confirming the Private BLE Device was added (or
explaining why it wasn't — most commonly "gave up" if Home Assistant's
scanner never saw the device broadcasting in range).

## Security note

The Long-Lived Access Token stored in `secrets.yaml` has full admin API
access, not just permission to add `private_ble_device` entries — Home
Assistant doesn't support narrower scopes for this API. Treat it with the
same care as any other admin credential, and don't commit `secrets.yaml`.
