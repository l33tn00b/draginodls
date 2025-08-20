# draginodls and sensecap T1000
Setting Up Dragino D-LS GPS Tracker and T1000-A Tracker for TTN


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



#  Linking it to fhem...
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

# Cobbling it to owntracks_recorder
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


