# draginodls
Setting Up Dragino D-LS GPS Tracker for TTN

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

- Linking it to fhem...
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
