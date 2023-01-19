# OMVS using MQTT and Home Assistant
## Overview
This guide shows you link your Open Vehicle Monitoring System (https://www.openvehicles.com/) to use your own MQTT server with TLS encryption, while also reading the data within Home Assistant.

## Preparing MQTT Broker
For the purpose of this section, I will be using a pre-existing Mosquitto MQTT Broker.

### Certificate Generation
#### Certificate Authority [CA]

```
cd /etc
cd mosquitto
mkdir certs
cd certs
mkdir ca
cd ca/
openssl req -new -x509 -days 365 -extensions v3_ca -keyout ca.key -out ca.crt
