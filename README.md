# draginodls and sensecap T1000
Setting Up Dragino D-LS GPS Tracker and T1000-A Tracker for TTN

When linking these to TTN, I use separate applications (because of separate payload decoders) (too lazy to create a unified one for both trackers).

## Dragino DLS
Just some quick notes:

- make sure to set channel SF12!

<img width="557" height="653" alt="grafik" src="https://github.com/user-attachments/assets/d1730d59-e69d-4018-9699-91d1389c7926" />
 
- Sending AT Commands is a pain:
<img width="578" height="392" alt="grafik" src="https://github.com/user-attachments/assets/0f39742e-21f2-493e-9260-e92db0fe57db" />

Single char input doesn't work. You need to paste the entire command at once into the terminal window. This is the intended behaviour. As per doc: 
```
7.8  Why when using some serial consoles, only inputting the first string port console will return "error"?

Need to enter the entire command at once, not a single character.
User can open a command window or copy the entire command to the serial console.
```

- What about the ACK?
From https://wiki.dragino.com/xwiki/bin/view/Main/End%20Device%20AT%20Commands%20and%20Downlink%20Command/:
<img width="951" height="318" alt="grafik" src="https://github.com/user-attachments/assets/2e136f39-f251-4a90-b9bb-8d6c89fc9685" />

This is much easier to understand...

- Upgrading the firmware
  - Get the ESP Download Tool from [https://www.dropbox.com/scl/fo/ztlw35a9xbkomu71u31im/AMBNyRAnzPS7eJJ1ibfl6WM/LoRaWAN%20End%20Node/TrackerD/Flash%20Tool?dl=0&rlkey=ojjcsw927eaow01dgooldq3nu](https://www.dropbox.com/scl/fo/ztlw35a9xbkomu71u31im/AMBNyRAnzPS7eJJ1ibfl6WM/LoRaWAN%20End%20Node/TrackerD/Flash%20Tool?rlkey=ojjcsw927eaow01dgooldq3nu&subfolder_nav_tracking=1&dl=0)
  - Head to [https://github.com/dragino/TrackerD/releases](https://github.com/dragino/TrackerD/releases) and get the latest firmware (yes, it's TrackerD, firmware of D and D-LS has been merged).
  - Unpack the firmware
  - The official wiki picture shows a download for a completely new device which has never been flashed. You don't need all the code shown in the image (see also: https://github.com/dragino/TrackerD/issues/45)
  - Make sure to choose ESP32 when starting the downloader
  - Download should look like this (after succesful completion):
    <img width="424" height="676" alt="grafik" src="https://github.com/user-attachments/assets/7645bd4b-d22c-4103-a0f4-a2f52936834a" />
  - Firmware (merged version) needs to be configured to know it is on a D-LS device: `Use AT+DEVICE=22 to configure as TrackerD-LS` as per https://github.com/dragino/TrackerD/issues/37 (and also changelog)
  - Well, with v1.5.6 the tracker shows a version of 1.0.7 instead of the factory 1.0.3 (not quite 1.5.6)...

###  Linking it to fhem...
Is doable. Create MQTT API key in TTN dashboard. MQTT username is app@ttn. MQTT password is API key. fhem somehow doesn't like MQTT over TLS. So we need a bridge in our MQTT broker (don't send unencrypted password over internet...).
mosquitto config file (e.g. `/etc/mosquitto/conf.d/bridge.conf`)
```
# new bridge
connection ttn
address eu1.cloud.thethings.network:8883
# alle topics, both directions
topic # both
# If set to true, all subscriptions and messages on the remote broker will be cleaned up
# if the connection drops. Note that setting to true may cause a large amount of retained
# messages to be sent each time the bridge reconnects.
cleansession true
start_type automatic
notifications false
local_password <pw of local broker>
local_username <username of local broker>
remote_username <app name>@ttn
remote_password <your api key>

# =================================================================
# Certificate based SSL/TLS support
# -----------------------------------------------------------------
#Path to the rootCA
# take the minimum ca file linkd in the ttn doc
bridge_cafile /etc/mosquitto/certs/rootCATTN.pem

# Path to the PEM encoded client certificate
# don't need this
#bridge_certfile /etc/mosquitto/certs/cert.crt

# Path to the PEM encoded client private key
# don't need this
#bridge_keyfile /etc/mosquitto/certs/private.key
```

## SenseCAP T1000-A (Seeed Studio)
Took me a while... but works.
- get the companion app (sensecraft)
- Skip registration in app (top right hand corner)
- expert setup (tap "resume")
- push the tracker's button for three seconds
- choose tracker in app
- wait for scan completion (less than a second)
- tap tracker id
- advanced configuration
- tab settings
- platform the things network
- frequency plan as per region
- activation type otaa
- app eui is the things network's join eui! (there is some strange note in the seeed wiki)
- DON'T tap the "copy" button for getting the IDs. The paste will include the id name (and therefore your device will never join the network). Instead mark the hex values and copy these for pasting into the ttn console!
- After going back to the app's first screen, the device will be disconnected and try joining the network.





# Cobbling Dragino to owntracks_recorder
TTN has an MQTT server. So does owntracks_recorder.
Since I've already got ot_recorder up and running, why not dump all the tracking data in there? Here we go...
Well, we need some glue logic for conversion... (mqtt -> http)

## create script
save as `home/pi/ttn2owntracks.py`
```
#!/usr/bin/env python3
import os, json, time, logging, signal, sys, re
from datetime import datetime, timezone
import requests
import paho.mqtt.client as mqtt
import socket
from urllib3.util.retry import Retry
from requests.adapters import HTTPAdapter


# ---- Configuration via env ----
MQTT_BROKER   = os.getenv("MQTT_BROKER", "localhost")
MQTT_PORT     = int(os.getenv("MQTT_PORT", "1883"))
MQTT_USER     = os.getenv("MQTT_USER")
MQTT_PASS     = os.getenv("MQTT_PASS")
# TTN-Topic: v3/<app>@ttn/devices/<device>/location/solved
# caveat "+" will dump all devices into one map
MQTT_TOPIC    = os.getenv("MQTT_TOPIC", "v3/<app name>@ttn/devices/+/location/solved")

# OwnTracks Recorder
RECORDER_URL  = os.getenv("RECORDER_URL", "<ip>:8083/pub")
OT_USER       = os.getenv("OT_USER", "ttn")      # à choissir
OT_DEVICE     = os.getenv("OT_DEVICE", "bridge") # à choissir
OT_TID        = os.getenv("OT_TID", "TT")        # two chars

REC_HTTP_USER = os.getenv("REC_HTTP_USER")
REC_HTTP_PASS = os.getenv("REC_HTTP_PASS")

# check host resolution
from urllib.parse import urlparse
host = urlparse(RECORDER_URL).hostname
try:
    socket.gethostbyname(host)
except Exception as e:
    # maybe we should do something more intelligent
    logging.error("RECORDER_URL Hostname not resolved: %s (%s)", host, e)
    sys.exit(2)

logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(message)s")
session = requests.Session()
session.headers.update({"Content-Type": "application/json"})
# Optional: per Header marking (Recorder accepted)
session.headers.update({"X-Limit-U": OT_USER, "X-Limit-D": OT_DEVICE})
auth = (REC_HTTP_USER, REC_HTTP_PASS) if REC_HTTP_USER and REC_HTTP_PASS else None

def _parse_received_at(ts: str) -> int:
    """
    TTN gives. '2025-08-11T14:54:10.727081707Z'
    -> in Unix-Seconds (int)
    """
    if not ts:
        return int(time.time())
    # 'Z' in +00:00 ; shorten ns to us
    ts = ts.replace("Z", "+00:00")
    # split fractional seconds, shorten to max 6  (Python does microseconds)
    m = re.match(r"^(.*\.\d{1,6})(\d+)([+-]\d{2}:\d{2})$", ts)
    if m:
        ts = m.group(1) + m.group(3)
    # Python 3.11: fromisoformat takes offset
    dt = datetime.fromisoformat(ts)
    return int(dt.timestamp())

def to_owntracks_from_ttn(msg: dict) -> dict:
    """
    TTN JSON (location/solved) -> OwnTracks location
    expects structure:
    {
      "end_device_ids": {"device_id": "...", ...},
      "received_at": "...",
      "location_solved": {"location": {"latitude": .., "longitude": .., "altitude":?, "accuracy":?}}
    }
    """
    dev = (msg.get("end_device_ids", {}) or {}).get("device_id", "")
    loc = (msg.get("location_solved", {}) or {}).get("location", {}) or {}
    lat = loc.get("latitude")
    lon = loc.get("longitude")
    if lat is None or lon is None:
        raise ValueError("TTN: no latitude/longitude in field location_solved.location")

    tst = _parse_received_at(msg.get("received_at"))
    device_name = OT_DEVICE

    ot = {
        "_type": "location",
        "lat": float(lat),
        "lon": float(lon),
        "tst": int(tst),
        "tid": (OT_TID or "TT")[:2],
        # optional, for debugging
        "topic": f"owntracks/{OT_USER}/{device_name}",
    }
    # optionale fields if applicable
    if "accuracy" in loc and loc["accuracy"] is not None:
        ot["acc"] = int(loc["accuracy"])
    if "altitude" in loc and loc["altitude"] is not None:
        ot["alt"] = float(loc["altitude"])
    return ot

def forward_to_recorder(payload: dict):
    url = f"{RECORDER_URL}?u={OT_USER}&d={OT_DEVICE}"
    r = session.post(url, data=json.dumps(payload), auth=auth, timeout=10)
    r.raise_for_status()
    logging.info("Recorder %s -> %s", r.status_code, r.text.strip() or "OK")

def on_connect(client, userdata, flags, rc, props=None):
    if rc == 0:
        logging.info("MQTT connected -> subscribe %s", MQTT_TOPIC)
        client.subscribe(MQTT_TOPIC, qos=1)
    else:
        logging.error("MQTT connect rc=%s", rc)

def on_message(client, userdata, msg):
    try:
        raw = msg.payload.decode("utf-8").strip()
        if not raw:
            return
        data = json.loads(raw)
        ot = to_owntracks_from_ttn(data)
        forward_to_recorder(ot)
    except Exception as e:
        logging.exception("Error at %s: %s", msg.topic, e)

def main():
    client = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2)
    if MQTT_USER:
        client.username_pw_set(MQTT_USER, MQTT_PASS)
    client.on_connect = on_connect
    client.on_message = on_message
    client.connect(MQTT_BROKER, MQTT_PORT, keepalive=60)

    def shutdown(*_):
        logging.info("Shutdown...")
        client.disconnect()
        sys.exit(0)

    for sig in (signal.SIGINT, signal.SIGTERM):
        signal.signal(sig, shutdown)

    client.loop_forever()

if __name__ == "__main__":
    main()
```


## Running the Service as the Default `pi` User

### Create Directory, Virtual Environment, and Script

```
# Work as pi user
sudo install -d -o pi -g pi /home/pi/ttn2owntracks
cd /home/pi/ttn2owntracks

# Create virtual environment and install dependencies
python3 -m venv .venv
. .venv/bin/activate
pip install --upgrade pip
pip install paho-mqtt requests

# Save script
nano /home/pi/ttn2owntracks/ttn2owntracks.py
# (Content = your current Python script)

deactivate
```

## env file
```
sudo tee /etc/ttn2owntracks.env >/dev/null <<'EOF'
# MQTT
MQTT_BROKER=<your broker ip>
MQTT_PORT=1883
MQTT_USER=<your broker user>
MQTT_PASS=<your broker password>
# caveat: if you have multiple trackers, this (+) will dump all of them into
# one map
MQTT_TOPIC=v3/<your application id>@ttn/devices/+/location/solved

# Recorder (HTTP only)
RECORDER_URL=http://your-hostname.example.tld:8083/pub
OT_USER=<owntracks user>
OT_DEVICE=<your device name>
# id doesn't matter
OT_TID=TT

# Optional Basic-Auth
# REC_HTTP_USER=owntracks
# REC_HTTP_PASS=supersecret
EOF
```
```
sudo chmod 640 /etc/ttn2owntracks.env
sudo chown root:root /etc/ttn2owntracks.env
```

## systemd file (runs as pi)
```
sudo tee /etc/systemd/system/ttn2owntracks.service >/dev/null <<'EOF'
[Unit]
Description=TTN -> OwnTracks Recorder bridge (MQTT in, HTTP out)
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=pi
Group=pi
WorkingDirectory=/home/pi/ttn2owntracks
Environment="PYTHONUNBUFFERED=1"
EnvironmentFile=-/etc/ttn2owntracks.env
ExecStart=/home/pi/ttn2owntracks/.venv/bin/python /home/pi/ttn2owntracks/ttn2owntracks.py
Restart=always
RestartSec=5

# Security hardening (adapted for home directory access)
NoNewPrivileges=yes
PrivateTmp=yes
ProtectSystem=full
ReadWritePaths=/home/pi/ttn2owntracks
RestrictAddressFamilies=AF_INET AF_INET6
SystemCallFilter=@system-service @network-io

[Install]
WantedBy=multi-user.target
EOF
```

##  Enable, Start, and Check Logs
```
sudo systemctl daemon-reload
sudo systemctl enable --now ttn2owntracks.service
sudo systemctl status ttn2owntracks.service
journalctl -u ttn2owntracks.service -f
```

## Updating the Service
### Edit the script
`nano /home/pi/ttn2owntracks/ttn2owntracks.py`

### Update Python packages
`. /home/pi/ttn2owntracks/.venv/bin/activate && pip install -U paho-mqtt requests && deactivate`

### Restart the service
`sudo systemctl restart ttn2owntracks.service`

# Cobbling SenseCap T1000 to owntracks_recorder
Pretty much the same as for Dragino D-LS with some slight variations...

## set up mqtt bridge
Maybe, if you already have an mqtt broker running locally

## create python script
ttn2owntracks_t1000.py, drop it into the same directory as the dragino one (i.e. /home/pi/ttn2owntracks). Use the same venv for both scripts.

```
import json
import os
import time
import logging
from typing import Any, Dict, Optional, Tuple, Iterable

import requests
from paho.mqtt import client as mqtt

# ---------- Configuration ----------
MQTT_HOST       = os.getenv("MQTT_HOST", "eu1.cloud.thethings.network")
# maybe, maybe we should default to tls
MQTT_PORT       = int(os.getenv("MQTT_PORT", "8883"))
# maybe, maybe we should default to tls
MQTT_TLS        = bool(int(os.getenv("MQTT_TLS", "1")))  # TTN uses TLS (8883)
MQTT_USER       = os.getenv("MQTT_USER", "")             # TTN: 'v3/<app-id>@ttn'
MQTT_PASS       = os.getenv("MQTT_PASS", "")             # TTN API-Key 'NNSXS...'
# will get data from all devices registered in an application ("+")
MQTT_TOPIC      = os.getenv("MQTT_TOPIC", "v3/+/devices/+/up")

RECORDER_BASEURL = os.getenv("RECORDER_BASEURL", "https://recorder.example.com/pub")
REC_HTTP_USER    = os.getenv("REC_HTTP_USER", "")        # Basic-Auth optional
REC_HTTP_PASS    = os.getenv("REC_HTTP_PASS", "")
TIMEOUT_SEC      = 10

logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(message)s")
session = requests.Session()

# ---------- Helpers ----------
def _flatten(xs: Any) -> Iterable[Any]:
    if isinstance(xs, list):
        for x in xs:
            yield from _flatten(x)
    else:
        yield xs

def derive_u_d_from_payload_or_topic(payload: Dict[str, Any], topic: str) -> Tuple[str, str]:
    # get user for owntracks recorder from payload or topic
    # 1) from payload  (TTN v3)
    app = (
        payload.get("end_device_ids", {})
        .get("application_ids", {})
        .get("application_id")
    )
    dev = payload.get("end_device_ids", {}).get("device_id")
    if app and dev:
        return app, dev
    # 2) fallback: from Topic v3/<app>@ttn/devices/<device>/up
    parts = topic.split("/")
    try:
        app_part = parts[1]               # '<app>@ttn'
        device   = parts[3]               # '<device>'
        app_id   = app_part.split("@")[0] # '<app>'
        return app_id, device
    except Exception:
        return "unknownapp", "unknowndevice"

def parse_lat_lon_tst_batt(payload: Dict[str, Any]) -> Tuple[float, float, int, Optional[int]]:
    """
    Extrahiert lat/lon/tst/batt aus TTN v3 Uplink-JSON.
    - Primär: decoded_payload.messages mit type 'Latitude'/'Longitude'
    - tst aus dortigem 'timestamp' (ms), sonst uplink_message.received_at, sonst now()
    - batt aus last_battery_percentage.value oder type 'Battery'
    """
    uplink = payload.get("uplink_message", {}) or {}
    dp = uplink.get("decoded_payload", {}) or {}

    lat = lon = None
    batt = None
    tst_ms = None

    # Prefer structured 'messages' list(s)
    msgs = dp.get("messages", None)
    if msgs is not None:
        for item in _flatten(msgs):
            if isinstance(item, dict):
                t = item.get("type")
                val = item.get("measurementValue")
                ts = item.get("timestamp")
                if t == "Latitude" and val is not None:
                    lat = float(val);  tst_ms = int(ts) if isinstance(ts, (int,float)) else tst_ms
                elif t == "Longitude" and val is not None:
                    lon = float(val);  tst_ms = int(ts) if isinstance(ts, (int,float)) else tst_ms
                elif t == "Battery" and val is not None and batt is None:
                    try: batt = int(round(float(val)))
                    except: pass

    # If still missing, give up early so caller can skip
    if lat is None or lon is None:
        raise ValueError("no lat/lon in decoded_payload.messages")

    # Timestamp fallback
    if tst_ms is None:
        rfc3339 = uplink.get("received_at") or payload.get("received_at")
        if isinstance(rfc3339, str):
            from datetime import datetime
            try:
                fmt = "%Y-%m-%dT%H:%M:%S.%fZ" if "." in rfc3339 else "%Y-%m-%dT%H:%M:%SZ"
                tst_ms = int(datetime.strptime(rfc3339, fmt).timestamp() * 1000)
            except: pass
    if tst_ms is None:
        import time as _t
        tst_ms = int(_t.time() * 1000)

    # Batt fallback from last_battery_percentage
    if batt is None:
        lb = uplink.get("last_battery_percentage", {})
        if isinstance(lb, dict) and "value" in lb:
            try: batt = int(round(float(lb["value"])))
            except: pass

    # Bounds check
    if not (-90 <= lat <= 90 and -180 <= lon <= 180):
        raise ValueError(f"unplausible lat/lon: {lat}, {lon}")

    return float(lat), float(lon), int(tst_ms // 1000), batt



def build_owntracks_payload(lat: float, lon: float, tst: int, batt: Optional[int]) -> Dict[str, Any]:
    ot = {"_type": "location", "lat": lat, "lon": lon, "tst": tst}
    if batt is not None:
        ot["batt"] = batt
    return ot

def post_to_recorder(u: str, d: str, ot_payload: Dict[str, Any]) -> None:
    url = f"{RECORDER_BASEURL}?u={u}&d={d}"
    headers = {"Content-Type": "application/json"}
    auth = (REC_HTTP_USER, REC_HTTP_PASS) if REC_HTTP_USER else None
    resp = session.post(url, data=json.dumps(ot_payload), headers=headers, auth=auth, timeout=TIMEOUT_SEC)
    if resp.status_code // 100 != 2:
        raise RuntimeError(f"Recorder HTTP {resp.status_code}: {resp.text}")

# ---------- MQTT Callbacks ----------
def on_connect(client, userdata, flags, rc, properties=None):
    if rc == 0:
        logging.info("MQTT connected")
        client.subscribe(MQTT_TOPIC, qos=1)
        logging.info(f"Subscribed: {MQTT_TOPIC}")
    else:
        logging.error(f"MQTT connect failed: rc={rc}")

def on_message(client, userdata, msg):
    try:
        payload = json.loads(msg.payload.decode("utf-8"))
        u, d = derive_u_d_from_payload_or_topic(payload, msg.topic)

        try:
            lat, lon, tst, batt = parse_lat_lon_tst_batt(payload)
        except Exception as e:
            # No coordinates in this frame → just log at INFO and return
            logging.info(f"Skipped (no lat/lon) u={u} d={d} reason={e}")
            return

        ot_payload = build_owntracks_payload(lat, lon, tst, batt)
        post_to_recorder(u, d, ot_payload)
        logging.info(f"→ Recorder OK  u={u} d={d} lat={lat:.6f} lon={lon:.6f} tst={tst} batt={batt}")
    except Exception as e:
        logging.exception(f"Fehler bei {msg.topic}: {e}")



def main():
    client = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2)
    if MQTT_USER or MQTT_PASS:
        client.username_pw_set(MQTT_USER, MQTT_PASS)
    if MQTT_TLS:
        client.tls_set()  # Standard-Truststore; bei Self-signed ggf. ca_certs angeben
    client.on_connect = on_connect
    client.on_message = on_message
    client.connect(MQTT_HOST, MQTT_PORT, keepalive=60)
    client.loop_forever()

if __name__ == "__main__":
    main()
```

## create systemd script
```
sudo tee /etc/systemd/system/ttn2owntracks_t1000.service >/dev/null <<'EOF'
[Unit]
Description=TTN -> OwnTracks Recorder Bridge for SenseCap T1000 (MQTT in, HTTP out)
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=pi
Group=pi
WorkingDirectory=/home/pi/ttn2owntracks
Environment="PYTHONUNBUFFERED=1"
EnvironmentFile=-/etc/ttn2owntracks_t1000.env
ExecStart=/home/pi/ttn2owntracks/.venv/bin/python /home/pi/ttn2owntracks/ttn2owntracks_t1000.py
Restart=always
RestartSec=5

# Security hardening (adapted for home directory access)
NoNewPrivileges=yes
PrivateTmp=yes
ProtectSystem=full
ReadWritePaths=/home/pi/ttn2owntracks
RestrictAddressFamilies=AF_INET AF_INET6
SystemCallFilter=@system-service @network-io

[Install]
WantedBy=multi-user.target
EOF
```

## create env file
```
cat > ttn2owntracks_t1000.env <<'EOF'
# --- MQTT / TTN ---
MQTT_HOST=your_mqtt_server
MQTT_PORT=8883
MQTT_TLS=1
MQTT_USER=mqtt_user
MQTT_PASS=mqtt_password
MQTT_TOPIC=v3/+/devices/+/up

# --- OwnTracks Recorder ---
RECORDER_BASEURL=http://example.owntracks_recorder:port/pub
REC_HTTP_USER=          # leave empty if no Basic-Auth
REC_HTTP_PASS=         # leave empty if no Basic-Auth
EOF

chmod 640 /etc/ttn2owntracks_t1000.env
chown root:root /etc/ttn2owntracks_t1000.env
```

## run it
```
systemctl --user daemon-reload
systemctl --user enable --now ttn2owntracks_t1000.service
systemctl status ttn2owntracks_t1000
```
