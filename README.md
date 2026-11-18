# Armbian Rock Pi 4B+ with Nebra LoRa HAT - Meshpoint -> Meshradar

Run the [meshpoint](https://github.com/KMX415/meshpoint) base station on a
**Radxa ROCK Pi 4B+** with the Nebra LoRa HAT, then optionally publish what it
hears to a **meshradar** map.

The HAT is the board known as the **Nebra Indoor Hotspot, Rock Pi version**,
carrying the **MAXIIOT GL5712-UX** module: SX1301 concentrator with two SX1257
radios, **915 MHz family — US915, AU915, KR920, AS923-1/2/3/4**.

Pinned at meshpoint `v0.7.9` (`main` `d7f2b7d`); this repo is `v0.7.9-sx1301`.

Upstream meshpoint supports SX1302/SX1303 concentrators only. SX1301 support
exists as a five-commit backend that is not merged, plus the fixes this board
needs. This repo ships them as six pinned patches against the official `main`, so
the install never depends on someone else's branch tip.

## Two names, two different things

| Name | What it is | Where it runs |
|---|---|---|
| **meshpoint** | the gateway: reads the concentrator, decodes packets, serves a dashboard, can uplink | on your Rock Pi |
| **meshradar** | a hosted map that receives gateway uplinks | on the internet |

Publishing to meshradar is optional. The **activation key is not**: meshpoint
verifies an `mr1_…` key offline at startup and exits without one, even when
nothing is published. Free key: <https://meshradar.io>.

This repo is the base install only. Regional radio settings — region, channel,
preset, MQTT broker — are deliberately **not** here; they ship as a separate
regional repo with an automated installer.

## Before you start

The HAT must already work. Do that first, and do not skip the verification steps —
a gateway on top of a deaf radio produces nothing but confusion:

- SPI enabled at 2 MHz → `/dev/spidev1.0`
- `nebra-lora-power` service holding POWER_EN
- `sx_probe.py` → `00=00 01=67 02=00 03=00`, `SX1301 ALIVE`
- `test_loragw_cal`: Rx A `143` (0x8F), Rx B `151` (0x97), no `WARNING` lines
  (one calibration command per radio, so those two differ — `0x9F` is the status
  `lgw_start` reports, not this test)

Full procedure, including the 40-pin pin map and the traps that cost us a day:
[rebuild-armbian-nebra-lora-hat-gl5712-ux — docs/hardware.md](https://github.com/fachos/rebuild-armbian-nebra-lora-hat-gl5712-ux/blob/main/docs/hardware.md#the-two-boards)
([pinout image](https://github.com/fachos/rebuild-armbian-nebra-lora-hat-gl5712-ux/blob/main/docs/assets/pinout-40pin.png)).

Also before you start:

- an `mr1_…` key from your meshradar account — the service will not stay up without it
- root on the Pi (`sudo -i`), because everything below writes under `/opt`

## SX1301 and SX1302 side by side

Upstream meshpoint targets SX1302/SX1303. On this HAT the concentrator is an
SX1301, so the backend talks to Semtech HAL 5.0.1 through a small native shim
instead of the SX1302 library:

| | SX1302 / SX1303 (upstream) | SX1301 (this install) |
|---|---|---|
| Python layer | `SX1302Wrapper` | `SX1301Wrapper` — same method names, selected by `capture.concentrator_backend: sx1301` |
| Native layer | `ctypes` → `libloragw-sx1302.so` | `ctypes` → `libsx1301_meshpoint.so` → HAL 5.0.1 |
| Receive modems | one standard modem + 8 multi-SF chains | one standard modem + 8 multi-SF chains that support **125 kHz only** |
| Presets at once | several SF/bandwidth combinations | **one** SF/bandwidth at a time on the standard modem |
| Spectral scan | yes, through the SX1261 companion radio | **no** |
| LBT / CAD / automatic relay | yes, through SX1261 | **no** — relay is refused by the backend |
| Noise floor | measured by scanning | derived from received packets |
| TX | standard modem, gain LUT | standard modem, gain LUT + per-region power cap |

What this gateway therefore cannot do on this HAT, and why it is not fixable in
software:

- **spectral scan** — needs the SX1261 companion radio that the GL5712-UX does not
  carry; the radio tab reports it unavailable instead of failing
- **LBT/CAD and automatic relay** — same missing companion; without a clear-channel
  check the backend refuses to key the transmitter on relayed traffic
- **two presets at once** — the standard modem holds one SF and one bandwidth, so a
  gateway hears MediumFast (SF9/250) *or* LongFast (SF11/250), never both. The eight
  multi-SF chains do not help here: they are 125 kHz only
- **VeryLongSlow (SF12/62.5 kHz)** — the SX1301 modems do not offer 62.5 kHz

## Install

Eight steps, in this order. Each one ends with what you must see before moving on.
Every path is absolute on purpose — none of these commands depend on where you
happen to be standing.

### 1. Packages

```bash
apt install -y build-essential git python3-venv python3-dev
python3 -c 'import venv, ensurepip; print("venv ok")'
```

### 2. The patch repo (this one)

```bash
cd /root
git clone https://github.com/fachos/meshpoint-nebra-lora-hat-gl5712-ux.git nebra-meshpoint
ls /root/nebra-meshpoint/patches/
```

You should see six patch files, applied in that order in step 4.

No git or no access on the Pi? The two files in `patches/` are all this repo
gives the machine — download the repo ZIP (**Code ▸ Download ZIP**), copy it over
(`scp meshpoint-*.zip root@<box>:/root/`), `unzip` it, and use that directory
wherever `/root/nebra-meshpoint` appears below. Nothing else in this repo is
needed on the box.

### 3. meshpoint itself, official `main`

```bash
git clone https://github.com/KMX415/meshpoint.git /opt/meshpoint
git -C /opt/meshpoint log --oneline -1
```

You should see `d7f2b7d …` — the commit both patches are pinned against. If it is
not, stop: `git -C /opt/meshpoint reset --hard d7f2b7d`.

### 4. Both patches

```bash
git -C /opt/meshpoint apply /root/nebra-meshpoint/patches/meshpoint-sx1301-backend.patch
git -C /opt/meshpoint apply /root/nebra-meshpoint/patches/sx1301_wrapper-rockpi.patch
git -C /opt/meshpoint apply /root/nebra-meshpoint/patches/meshpoint-ui-no-flicker.patch
git -C /opt/meshpoint apply /root/nebra-meshpoint/patches/meshpoint-no-selfupdate-on-sx1301.patch
git -C /opt/meshpoint apply /root/nebra-meshpoint/patches/meshpoint-identity-and-uplink.patch
git -C /opt/meshpoint apply /root/nebra-meshpoint/patches/meshpoint-region-plan.patch
git -C /opt/meshpoint status --short | head
```

`git apply` prints nothing on success. `status` should list the new SX1301 files.

| Patch | What it is |
|---|---|
| `meshpoint-sx1301-backend.patch` | the SX1301 support: `src/hal/sx1301_wrapper.py`, `src/capture/sx1301_source.py`, `scripts/sx1301_native.c` + the native build and provenance scripts. Diff of `flynnr2`'s `submission/sx1301-backend-20261025` (tip `24d1d4b`, 5 commits) against `main` `d7f2b7d`. Verified: `git apply --check` clean on `d7f2b7d`. |
| `sx1301_wrapper-rockpi.patch` | this board: `reset_gpio` range `0–27` → `0–1023`, `pinctrl set …` → sysfs fallback, US-only band gate → per-region bands, and errors that name the cause: unknown region and wrong sync word are separate messages, and TX reports receiver-not-configured vs. parameters that differ instead of one generic line. Applies on top of the backend patch. |
| `meshpoint-ui-no-flicker.patch` | the dashboard's node list rebuilt its entire DOM on every packet and on every 15 s poll, so the panel flickered continuously; it now re-renders only when the markup changes. Applies on top of the other two. |
| `meshpoint-no-selfupdate-on-sx1301.patch` | the dashboard's updater runs `git reset --hard origin/<branch>`, which would delete these patches and leave a gateway that cannot open the concentrator. On this backend it refuses with a message instead. Applies on top of the others. |
| `meshpoint-identity-and-uplink.patch` | advertises a chosen `HardwareModel` and device `Role`, stores the role it decodes from peers, and makes MQTT uplink actually work: own transmissions are published too, and every uplink carries the ciphertext. Meshtastic firmware with MQTT encryption on discards decoded packets (`Ignore decoded message on MQTT, encryption is enabled`), which is why a stock meshpoint looks silent to the rest of the mesh. The MQTT card's topic preview is derived from `publish_channels` instead of a hardcoded `LongFast`. Applies on top of the others. |
| `meshpoint-region-plan.patch` | `radio.region` becomes a real plan: `MY_919` exists, the dashboard dropdown is built from the region table (band, power limit, duty cycle), and the carrier is derived like the firmware does when `frequency_mhz` is omitted — `start + bw/2 + slot`, slot from the channel name. Applies on top of the others. |

### 5. Python environment

```bash
python3 -m venv /opt/meshpoint/venv
/opt/meshpoint/venv/bin/pip install -U pip
/opt/meshpoint/venv/bin/pip install -r /opt/meshpoint/requirements.txt
/opt/meshpoint/venv/bin/python -c 'import fastapi, uvicorn, yaml; print("deps ok")'
```

### 6. The native shim (this is what talks to the SX1301)

The HAL is the one the hardware repo already built — no second checkout, no SPI
patch here. `build_sx1301_native.sh` refuses a dirty tree, so the SPI change in
`/opt/lora_gateway` is committed first:

```bash
grep -E '^#define SPI_(SPEED|DEV_PATH)' /opt/lora_gateway/libloragw/src/loragw_spi.native.c
#   #define SPI_SPEED       2000000
#   #define SPI_DEV_PATH    "/dev/spidev1.0"

git -C /opt/lora_gateway -c user.name="spi patch" -c user.email="spi@localhost" \
    commit -qam "rock pi: spidev1.0 at 2 MHz"

/opt/meshpoint/scripts/build_sx1301_native.sh /opt/lora_gateway
```

You should see:

```
Built /opt/meshpoint/build/libsx1301_meshpoint.so (native ABI 2)
```

If it stops with `HAL tracked files must be clean`, the commit above did not
happen. If you re-run the hardware repo's `build-hal.sh` later it resets that
checkout — commit the SPI change again before rebuilding the shim.

### 7. Configuration

```bash
install -d -m 700 /opt/meshpoint/data
install -m 600 /root/nebra-meshpoint/config/local.yaml.example /opt/meshpoint/config/local.yaml
```

Now edit exactly three values in `/opt/meshpoint/config/local.yaml`:

| Key | Set it to |
|---|---|
| `device.device_id` | 8 hex characters, unique to this gateway: `openssl rand -hex 4` |
| `upstream.auth_token` | your `mr1_…` key — **required**, the service exits without it |
| `capture.sx1301_lib_path` | leave as `/opt/meshpoint/build/libsx1301_meshpoint.so` |
| `device.device_id` | set it once (`openssl rand -hex 16`). Left empty, meshpoint generates an ephemeral id per start — that is upstream behaviour, and the node id follows it |
| `device.hardware_model` | what NodeInfo advertises: `37` PORTDUINO (generic Linux host, the honest default), `43` HELTEC_V3, `9` RAK4631, `255` PRIVATE_HW |
| `mqtt.publish_channels` | must list the channel name your region uses (`MediumFast` for MY_919) — gate 2 blocks any channel not listed |
| `mqtt.gateway_id` | the dashboard shows this node as `!xxxxxxxx`; paste that value here once so broker uplinks are attributed to the node. Left empty, meshpoint falls back to a hash of `device_name` (upstream default) |
| `device.role` | mesh role: `CLIENT`, `CLIENT_MUTE`, `ROUTER`, `ROUTER_CLIENT` (use this for a fixed gateway), `REPEATER`, `TRACKER`, `SENSOR`, `CLIENT_HIDDEN`, `LOST_AND_FOUND` |

Everything else can stay: `transmit`, `mqtt` and `upstream.enabled` are already
`false`, so nothing keys a transmitter or publishes before you have proven the
receive path.

```bash
grep -E 'device_id|auth_token|sx1301_lib_path' /opt/meshpoint/config/local.yaml
test -f /opt/meshpoint/build/libsx1301_meshpoint.so && echo "shim present"
```

Radio parameters live in the same file. Omit `frequency_mhz` (or set it to
`null`) and the carrier is derived from the region plan, the bandwidth and the
channel name — the same arithmetic the Meshtastic firmware uses, so picking a
region and a preset is enough. Set it explicitly only to pin a carrier.

Carrier, spreading factor and bandwidth must match the network you intend to
listen to:

```yaml
radio:
  region: US                 #  ← your regional plan
  frequency_mhz: 906.875     #  ← your channel centre
  spreading_factor: 11
  bandwidth_khz: 250
  coding_rate: 4/5
  sync_word: 43              # 0x2B, Meshtastic
```

Guard rails in the code, so a typo cannot key a transmitter outside the band the
hardware is allowed to use:

- unsupported `region` → the backend lists the regions it accepts
- carrier outside that band → `SX1301 channel extends outside the … band`

### Choosing the carrier

Radio parameters are per country, and this repo deliberately ships only the US
example above. The carrier is not free-form either: Meshtastic derives it from the
region table and the primary channel name, so take the carrier, the preset (which
fixes SF and bandwidth) and the sync word from your regional plan.

Region names the backend knows: `US`, `MY_919` (or `MY`), `SG_923` (or `SG`). TX
power is capped by region as well as by the gain table — US 30 dBm, MY_919 27 dBm,
SG_923 20 dBm — so a high `tx_power_dbm` cannot exceed the local limit.

The backend enforces the band it knows: it refuses a carrier whose occupied
bandwidth would leave the region's band with `SX1301 channel extends outside the …
band`. A band edge is not a carrier — 250 kHz centred on the edge spends half its
width outside.

### Region plan: `MY_919`

`radio.region: MY_919` is Malaysia, 919–924 MHz, 27 dBm limit, no duty cycle. The
carrier is not a single number — each preset has its own, derived with
`start + bw/2 + slot` where the slot comes from the channel name (a blank channel
name resolves to the preset name):

| Preset | SF | Bandwidth | Coding rate | Carrier |
|---|---|---|---|---|
| VeryLongSlow | SF12 | 62.5 kHz | 4/8 | 922.031 MHz (not supported by SX1301) |
| ShortSlow | SF8 | 125 kHz | 4/8 | 920.312 MHz |
| LongModerate | SF10 | 125 kHz | 4/8 | 923.812 MHz |
| MediumSlow | SF10 | 125 kHz | 4/8 | 921.438 MHz |
| LongSlow | SF12 | 125 kHz | 4/8 | 922.312 MHz |
| ShortFast | SF7 | 250 kHz | 4/5 | 923.875 MHz |
| **MediumFast** | **SF9** | **250 kHz** | **4/5** | **921.125 MHz** |
| LongFast | SF11 | 250 kHz | 4/5 | 922.875 MHz |
| ShortTurbo | SF7 | 500 kHz | 4/5 | 921.750 MHz |
| LongTurbo | SF11 | 500 kHz | 4/8 | 919.750 MHz |

The same arithmetic is used by the Meshtastic firmware, so picking region, preset
and channel name is enough; the numbers above are what the gateway computes. The
channel plan itself lives in its own repo:
[meshtastic-my919](https://github.com/fachos/meshtastic-my919).

### 8. Service

```bash
cat >/etc/systemd/system/meshpoint.service <<'EOF'
[Unit]
Description=meshpoint gateway (SX1301)
After=network-online.target nebra-lora-power.service
Wants=network-online.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/meshpoint
Environment=CONCENTRATOR_CONFIG=/opt/meshpoint/config/local.yaml
ExecStart=/opt/meshpoint/venv/bin/python -m uvicorn src.api.server:create_app --factory --host 0.0.0.0 --port 8080 --app-dir /opt/meshpoint
Restart=always
RestartSec=5
SyslogIdentifier=meshpoint

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable --now meshpoint
systemctl is-active meshpoint
```

`WorkingDirectory=/opt/meshpoint` plus `--app-dir /opt/meshpoint` are not optional:
the app imports `src.*` from there, and `CONCENTRATOR_CONFIG` is what makes it read
your config instead of a default. `--app-dir` is also why the same command works
from any directory when you run it by hand.

Identity is config, not code. Three values, set once:

```
device.device_id   openssl rand -hex 16  -> written to local.yaml once
transmit.node_id   left empty; derived from device_id, then persisted
mqtt.gateway_id    paste the !xxxxxxxx the dashboard shows for this node
```

Leaving them empty is the upstream default: `device_id` is re-generated per start
and `gateway_id` becomes a hash of `device_name`. Both work; neither is what a
public map expects, so this install fills them in.

Transport is not a config key: `via_mqtt` is a per-packet flag set by whoever
carries it. This gateway hears LoRa and publishes to MQTT, so its own packets show
"LoRa" to a neighbour and "MQTT" to everyone reached through the broker.

## Check that it is actually running

```bash
systemctl is-active meshpoint
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8080/
journalctl -u meshpoint -n 25 --no-pager
```

Expected: `active`, then **`302`** (that is the redirect to `/setup`), then a
banner like this:

```
   Device       Nebra LoRa HAT gateway (3f9a12c4)
   Version      0.7.9
   Source       concentrator (SX1301 single modem)
   Frequency    906.875 MHz / SF11 / BW250 (US)
   Upstream     disabled
   Dashboard    http://192.168.99.240:8080

Meshpoint started -- listening for packets
```

`302` and not `200` on the first visit is correct: nothing is configured for the
browser yet. `lgw_connect: INFO: no FPGA detected or version not supported (v0)`
is normal on SX1301 — ignore it.

Open `http://<box-ip>:8080`, set the admin password on `/setup`, then log in.
`/api/...` answering `401` before that is also normal.

To watch it in the foreground, exactly as the unit runs it (Ctrl-C to stop):

```bash
CONCENTRATOR_CONFIG=/opt/meshpoint/config/local.yaml \
/opt/meshpoint/venv/bin/python -m uvicorn src.api.server:create_app \
  --factory --host 0.0.0.0 --port 8080 --app-dir /opt/meshpoint
```

## Uplink to meshradar (optional)

The key from step 7 activates the gateway. To publish as well:

```yaml
upstream:
  enabled: true
  auth_token: mr1_REPLACE_WITH_YOUR_KEY
```

```bash
systemctl restart meshpoint
journalctl -u meshpoint -f | grep -i upstream
```

```
-- UPSTREAM connecting to wss://api.meshradar.io
-- UPSTREAM connecting  auth=present
-- UPSTREAM connected to wss://api.meshradar.io
```

Only packets your radio actually receives are published. Set `device.device_name`
to something you are happy to see on a public map before enabling this.

## Turning on transmit

Once you have heard something, or a handheld sees the gateway:

```yaml
transmit:
  enabled: true
  tx_power_dbm: 10            # start here; raise only after a range check
  hop_limit: 3
```

`tx_power_dbm` must be one of the rows in `capture.sx1301_tx_gain_lut` — with the
shipped table that is `-6, -3, 0, 3, 6, 10, 11, 12, 13, 14, 16, 20, 23, 25, 26, 27`.
29 or 30 is not a row and cannot be produced by this PA: the backend clamps to the
nearest lower row and logs one warning instead of failing every broadcast.

```bash
systemctl restart meshpoint
journalctl -u meshpoint -n 30 --no-pager | grep -i '^.*TX'
```

```
TX NodeInfo: src=... len=94
NodeInfo broadcast OK: airtime=283ms
```

Never key the PA without an antenna. `TX requires the verified carrier gain
table` means the gain LUT was not loaded — leave transmit off and fix that first.

TX shares the receiver's radio chain, so it cannot work while the concentrator is
down: that is what `SX1301 TX unavailable: the receiver is not configured` says.
Fix capture first, then transmit.

## Troubleshooting

| Symptom | Cause |
|---|---|
| dashboard shows only fetch errors | the service is dead, not the browser: `systemctl is-active meshpoint`, then `journalctl -u meshpoint -n 30` |
| `Meshpoint is not activated` / `Invalid API key` | `upstream.auth_token` empty or wrong — a key is required even for a local-only gateway |
| `Capture source concentrator failed to start` | the traceback below it names the call: `sx1301_lib_path` wrong, the `.so` never built (step 6), `/dev/spidev1.0` missing, or `configure()` refused the radio |
| `… has no band for region '…'` / `needs the Meshtastic sync word 0x2B (43)` / `channel extends outside the … band` | the radio block does not match the region: region name, sync word, or a carrier whose bandwidth leaves the band |
| `SX1301 TX unavailable: the receiver is not configured` | transmit is on while the concentrator is down — TX and RX share one radio chain, so fix capture first |
| `SX1301 TX must match configured receiver: packet … receiver …` | the two tuples in the message are the packet and the live receiver; fix the field that differs |
| `ModuleNotFoundError: No module named 'src'` | started without `--app-dir /opt/meshpoint` — copy the unit's `ExecStart` line verbatim |
| `/api/...` returns 401 / dashboard 302 loop | not logged in yet; finish `/setup`, then reload |
| `HAL tracked files must be clean` | the SPI edit in `/opt/lora_gateway` was not committed — see step 6 |
| every register `0x66` | chip in reset or unpowered — see the hardware repo |
| calibration `0x00` | TCXO not settled; cold boot, check SPI speed |
| messages typed on the gateway only reach RF neighbours | MQTT uplinks must carry the ciphertext — see the identity-and-uplink patch; a decoded envelope is dropped by firmware |
| 0 packets for hours on a known-active channel | carrier, preset (SF + bandwidth) or sync word does not match the network — frequency alone is not enough |

Do not run `meshpoint setup` on this board. The wizard looks for
`/dev/spidev0.x` and writes an SX1302 capture block; this HAT is `spidev1.0` with
the SX1301 backend, so the wizard reports no hardware and would overwrite a
working config. `local.yaml` above is the supported path here.

## Licence and attribution

- Gateway software: **meshpoint** by KMX415, AGPL-3.0 —
  <https://github.com/KMX415/meshpoint>, pinned here at `main` `d7f2b7d` (v0.7.9)
- SX1301 backend: `submission/sx1301-backend-20261025` by **flynnr2**
  (tip `24d1d4b`, 5 commits), shipped as `patches/meshpoint-sx1301-backend.patch`
- HAL: **Semtech** `Lora-net/lora_gateway` 5.0.1
- This board's bring-up (SPI overlay, POWER_EN, pin map, calibration):
  [rebuild-armbian-nebra-lora-hat-gl5712-ux](https://github.com/fachos/rebuild-armbian-nebra-lora-hat-gl5712-ux)
- This repo: docs MIT, patch files AGPL-3.0 (they are derivative works)

Project version: **v0.7.9-sx1301** — meshpoint v0.7.9 plus the patches in this
repo, applied in the order given in step 4.

If you distribute an image or bundle containing meshpoint, the AGPL requires you
to publish the corresponding source.

## Support

If this saved you time, you can put a little back — it goes into bench hardware
and more notes like these.

- **Ko-fi** — <https://ko-fi.com/fachos>
- **BTC** — `3GXWxonjriTtboDWDvTTBv3VJ1nhvv3fMW`
- **SOL** — `HbDMoGWS1Kj23qSfon9XK2F9rzbdFmh7nwsnB5j8TXup`

Crypto transfers are irreversible — verify every character before sending.
