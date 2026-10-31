# ReThink Cloud add-on (unofficial fork)

Runs the [jmcjm/rethink](https://github.com/jmcjm/rethink) fork (`deploy` branch) of
[anszom/rethink](https://github.com/anszom/rethink) — a fully-local cloud server for
LG ThinQ devices — directly on Home Assistant OS. On top of upstream, the fork adds a
handler for the RH90V9_WW heat-pump dryer and extended remote control for the
Y_V8_Y___W.B32QEUK washer (EU). Not affiliated with upstream.

## How it works

LG devices resolve `common.lgthinq.com`; your router must hijack that DNS name to the
Home Assistant host IP (and DNAT stray DNS/NTP from the IoT VLAN). The add-on runs with
host networking and binds:

| Port  | Role                          |
| ----- | ----------------------------- |
| 443   | HTTPS (ThinQ2 API)            |
| 8884  | MQTTS (device connection)     |
| 1884  | plain MQTT (local debugging)  |
| 46030 | ThinQ1 HTTPS                  |
| 47878 | ThinQ1 plain                  |
| 44401 | management panel              |

The management panel opens from the **ReThink** entry in the Home Assistant sidebar and from
the **Open Web UI** button, through Ingress, and stays available at `http://<ha-host>:44401`.
Ingress is fixed to port 44401, so changing `management_port` leaves only the direct address
working.

## Configuration

Everything routine is configured from the add-on's **Configuration tab**: hostname,
MQTT broker URL and credentials, discovery/rethink prefixes, all ports, and log
topics. On every start the add-on regenerates `config.json` from these options.

Set `manual_config: true` if you need config features the UI does not expose (e.g.
split `bind`/`advertise` ports) — the add-on then leaves `config.json` alone and you
edit it by hand in the configuration directory.

Files in the add-on configuration directory
(`/addon_configs/<slug>_rethink/`, mapped to `/config` inside the container):

```
config.json   generated from the UI options (or hand-written with manual_config)
ca.key        CA private key   — losing this forces re-provisioning of all devices
ca.cert       CA certificate
state/        bridge state (oauth2 token, device registrations)
```

`mqtts_port` defaults to 8884 because the Mosquitto add-on usually occupies 8883 on
the host. rethink advertises this port to devices dynamically, so any free port works.

### Migrating from an existing rethink instance

1. Install the add-on but do not start it yet.
2. Bring over your CA — two ways:
   - **UI only:** in the Configuration tab (three-dot menu → Edit in YAML) paste the
     contents of your existing files into `ca_key_pem` and `ca_cert_pem` as YAML
     block scalars:

     ```yaml
     ca_key_pem: |
       -----BEGIN PRIVATE KEY-----
       ...
     ca_cert_pem: |
       -----BEGIN CERTIFICATE-----
       ...
     ```

     On start the add-on writes them to `ca.key` / `ca.cert`. While these options
     are set they overwrite the files on every start; you can clear them afterwards
     — the files stay.
   - **Files:** copy `ca.key` and `ca.cert` into `/addon_configs/<slug>_rethink/`
     (Samba/SSH add-on or the Studio Code Server add-on).
3. If you used bridge mode (ThinQ app access via the real LG cloud), also copy your
   `state/` directory the file way — it cannot be pasted through the UI. Without it
   devices still connect locally; only the bridge registrations are lost.
4. In the Configuration tab set **`hostname` to the CN of your existing `ca.cert`**
   (for old installs this is typically the LAN IP the devices were provisioned
   against). If the hostname stops matching the certificate, the server generates a
   fresh CA and every LG device has to be re-provisioned through its SoftAP.
5. Point `mqtt_url` at the local broker (e.g. `mqtt://localhost:1883`) and fill in
   `mqtt_user` / `mqtt_pass`.
6. Start the add-on and watch the log; devices should reconnect within a minute or
   two of their next DNS lookup.

## Notes

- `boot` is `manual` by default — switch it to auto once you have verified the setup.
- Port 443 must be free on the host; HAOS itself does not use it, but another add-on
  (e.g. NGINX proxy) might.
