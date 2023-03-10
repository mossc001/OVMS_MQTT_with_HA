# OVMS using MQTT and Home Assistant
## Overview
This guide shows you link your Open Vehicle Monitoring System (https://www.openvehicles.com/) to use your own MQTT server with TLS encryption, while also reading the data within Home Assistant.

## Preparing MQTT Broker
For the purpose of this section, I will be using a pre-existing Mosquitto MQTT Broker. Credit to https://openest.io/ for the majority of the content for the Certificate Generation, slightly modified for my purpose.

### Certificate Generation
#### Certificate Authority [CA]
In the console of your broker, use the following commands:
```
$ cd /etc
$ cd mosquitto
$ mkdir certs
$ cd certs
$ mkdir ca
$ cd ca/
$ openssl req -new -x509 -days 365 -extensions v3_ca -keyout ca.key -out ca.crt
Generating a RSA private key
.....+++++
................................+++++
writing new private key to 'ca.key'
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
-----
```
Once you have created the PEM pass phrase, you will be required to enter some dummy details for the CA. Note that it's best not to reference any existing hostname or FQDN in the 'Common Name':
```
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:GB
State or Province Name (full name) []:Oxfordshire
Locality Name (eg, city) [Default City]:Oxford
Organization Name (eg, company) [Default Company Ltd]:MossC Ltd
Organizational Unit Name (eg, section) []: IT Dept
Common Name (eg, your name or your server's hostname) []: MossC Ltd
Email Address []:
```
Check your certificates and keys are there, then go up a directory.
```
$ ls
$ ca.crt  ca.key
$ cd ..
```
#### Certificate for MQTT Broker
Now we have a certificate. We can create the keys and certificates of the broker. A private key is generated (without a password):
```
$ mkdir broker
$ cd broker
$ openssl genrsa -out broker.key 2048
Generating RSA private key, 2048 bit long modulus (2 primes)
.................................................................................................................................+++++
.......................................................................................+++++
e is 65537 (0x010001)
```
Now, we create a signing request file from this key :
```
$ openssl req -out broker.csr -key broker.key -new
```
You will now be asked to enter key details relating to the broker certificate. It is very important that the Common Name is correct otherwise your certificate will fail to authenticate from a client.
In my setup, to hit my broker externally, it's from mqtt.cheese.com (fake example), therefore this will be the Common Name:
```
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:GB
State or Province Name (full name) []:Oxfordshire
Locality Name (eg, city) [Default City]:Oxford
Organization Name (eg, company) [Default Company Ltd]:MossC Ltd
Organizational Unit Name (eg, section) []: IT Dept
Common Name (eg, your name or your server's hostname) []: mqtt.cheese.com
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```
Check your certificates and keys are there.
```
$ ls
broker.csr  broker.key
```
Now we can pass the Certificate Signing Request (csr) file to our validation authority:
```
$ openssl x509 -req -in broker.csr -CA ../ca/ca.crt -CAkey ../ca/ca.key -CAcreateserial -out broker.crt -days 100
Signature ok
subject=C = FR, ST = France, L = Strasbourg, O = opeNest, CN = localhost, emailAddress = contact@openest.io
Getting CA Private Key
Enter pass phrase for ../ca/ca.key:
```
Check your certificates and keys are there.
```
$ ls
broker.crt  broker.csr  broker.key
```
#### Certificate for Client
A certificate for the client is not required for OVMS as it only needs to have the CA certificate on the module to validate the broker certificate.
If you do want to generate a client certificate, follow the same process as 'Certificate for MQTT Broker', but change the Common Name to the hostname of the client.

### MQTT Configuration File Modifications
As we don't want to put client certificates on all our existing MQTT devices that may be on our local network, but we do want TLS for external, we must split the MQTT configuration file by listeners/ports.
We want the following:
- Port 1883 - Internal only without encryption and no password (my setup)
- Port 8883 - External only with encryption and password
Further to the above, we must also reference our certificates generated in the prior steps. If you kept the pathes the same as above, the configuration path for the certificates will be correct in the below:
```
$ nano /etc/mosquitto/mosquitto.conf
```
```
# Place your local configuration in /etc/mosquitto/conf.d/
#
# A full description of the configuration file is at
# /usr/share/doc/mosquitto/examples/mosquitto.conf.example

pid_file /run/mosquitto/mosquitto.pid

persistence true
persistence_location /var/lib/mosquitto/

log_dest file /var/log/mosquitto/mosquitto.log

include_dir /etc/mosquitto/conf.d

per_listener_settings true

# Per listener 
port 1883
allow_anonymous true

listener 8883
cafile /etc/mosquitto/certs/ca/ca.crt
certfile /etc/mosquitto/certs/broker/broker.crt
keyfile /etc/mosquitto/certs/broker/broker.key
#require_certificate true
allow_anonymous false
password_file /etc/mosquitto/passwordfile
tls_version tlsv1.2
```
## Preparing OVMS for MQTT & Certificates
### Certificate Authority Import
In order for OVMS to connect to MQTT with TLS, it must have the Certificate Authority within its trusted certificate list. If you go to 'Tools' > 'Shell' and input the command below, it will list the trusted Certificate Authorities currently on the module:
```
tls trust list
```
To put our own Certificate Authority on the OVMS module, we must first create a folder, then create a file for the raw Certificate Authority data. A folder can be created by going to 'Tools' > 'Editor', then pressing the 'New dir' button. The new folder should be called "trustedca" and be a sub-folder of "store":
```
/store/trustedca/
```
Next you need to input your Certificate Authority data into the editor. Simply open your "ca.crt" created in the prior steps in NotePad, and copy/paste the complete contents into the OVMS editor. Then press "Save As" and save it to the location '/store/trustedca/' with a filename ending in .pem e.g. mqtt.pem.
```
-----BEGIN CERTIFICATE-----
MIIFazCCA1OgAwIBAgIRAIIQz7DSQONZRGPgu2OCiwAwDQYJKoZIhvcNAQELBQAw
TzELMAkGA1UEBhMCVVMxKTAnBgNVBAoTIEludGVybmV0IFNlY3VyaXR5IFJlc2Vh
cmNoIEdyb3VwMRUwEwYDVQQDEwxJU1JHIFJvb3QgWDEwHhcNMTUwNjA0MTEwNDM4
WhcNMzUwNjA0MTEwNDM4WjBPMQswCQYDVQQGEwJVUzEpMCcGA1UEChMgSW50ZXJu
ZXQgU2VjdXJpdHkgUmVzZWFyY2ggR3JvdXAxFTATBgNVBAMTDElTUkcgUm9vdCBY
MTCCAiIwDQYJKoZIhvcNAQEBBQADggIPADCCAgoCggIBAK3oJHP0FDfzm54rVygc
h77ct984kIxuPOZXoHj3dcKi/vVqbvYATyjb3miGbESTtrFj/RQSa78f0uoxmyF+
0TM8ukj13Xnfs7j/EvEhmkvBioZxaUpmZmyPfjxwv60pIgbz5MDmgK7iS4+3mX6U
A5/TR5d8mUgjU+g4rk8Kb4Mu0UlXjIB0ttov0DiNewNwIRt18jA8+o+u3dpjq+sW
T8KOEUt+zwvo/7V3LvSye0rgTBIlDHCNAymg4VMk7BPZ7hm/ELNKjD+Jo2FR3qyH
B5T0Y3HsLuJvW5iB4YlcNHlsdu87kGJ55tukmi8mxdAQ4Q7e2RCOFvu396j3x+UC
B5iPNgiV5+I3lg02dZ77DnKxHZu8A/lJBdiB3QW0KtZB6awBdpUKD9jf1b0SHzUv
KBds0pjBqAlkd25HN7rOrFleaJ1/ctaJxQZBKT5ZPt0m9STJEadao0xAH0ahmbWn
OlFuhjuefXKnEgV4We0+UXgVCwOPjdAvBbI+e0ocS3MFEvzG6uBQE3xDk3SzynTn
jh8BCNAw1FtxNrQHusEwMFxIt4I7mKZ9YIqioymCzLq9gwQbooMDQaHWBfEbwrbw
qHyGO0aoSCqI3Haadr8faqU9GY/rOPNk3sgrDQoo//fb4hVC1CLQJ13hef4Y53CI
rU7m2Ys6xt0nUW7/vGT1M0NPAgMBAAGjQjBAMA4GA1UdDwEB/wQEAwIBBjAPBgNV
HRMBAf8EBTADAQH/MB0GA1UdDgQWBBR5tFnme7bl5AFzgAiIyBpY9umbbjANBgkq
hkiG9w0BAQsFAAOCAgEAVR9YqbyyqFDQDLHYGmkgJykIrGF1XIpu+ILlaS/V9lZL
ubhzEFnTIZd+50xx+7LSYK05qAvqFyFWhfFQDlnrzuBZ6brJFe+GnY+EgPbk6ZGQ
3BebYhtF8GaV0nxvwuo77x/Py9auJ/GpsMiu/X1+mvoiBOv/2X/qkSsisRcOj/KK
NFtY2PwByVS5uCbMiogziUwthDyC3+6WVwW6LLv3xLfHTjuCvjHIInNzktHCgKQ5
ORAzI4JMPJ+GslWYHb4phowim57iaztXOoJwTdwJx4nLCgdNbOhdjsnvzqvHu7Ur
TkXWStAmzOVyyghqpZXjFaH3pO3JLF+l+/+sKAIuvtd7u+Nxe5AW0wdeRlN8NwdC
jNPElpzVmbUq4JUagEiuTDkHzsxHpFKVK7q4+63SM1N95R1NbdWhscdCb+ZAJzVc
oyi3B43njTOQ5yOf+1CceWxG1bQVs5ZufpsMljq4Ui0/1lvh+wjChP4kqKOJ2qxq
4RgqsahDYVvTH9w7jXbyLeiNdd8XM2w9U/t7y0Ff/9yi0GE44Za4rF2LN9d11TPA
mRGunUHBcnWEvgJBQl9nJEiU0Zsnvgc/ubhPgXRR4Xq37Z0j4r7g1SgEEzwxA57d
emyPxgcYxn/eR44/KJ4EBs+lVDR3veyJm+kXQ99b21/+jh5Xos1AnX5iItreGCc=
-----END CERTIFICATE-----
```
Once saved, enter the below or manually reboot the OVMS module:
```
tls trust reload
```
To validate if your Certificate Authority has been loaded, simply enter the below and look for your CA. If not, double check the above steps.
```
tls trust list
```
## Configuring OVMS Server V3 (MQTT)
### Server V3 (MQTT) configuration
Navigate to Server V3 (MQTT) configuration page 'Config' > 'Server V3 (MQTT)', and you will be presented with the standard MQTT configuration screen.
Input the specifics of your MQTT server, see example below:
```
Server: mqtt.cheese.com
Enable TLS: ENABLED / TICKED
Port: 8883
Username: YOUR SETUP USERNAME (Not in this guide)
Password: YOUR SETUP PASSWORD (Not in this guide)
```
You should setup the intervals as fit or use the defaults.
Once setup, you will need need to restart Server V3 (MQTT) by pressing 'Stop V3', then 'Start V3' from the 'Status' menu.

## Verifying The MQTT Messages
I would recommend verying the messages using a client such as MQTT Explorer (http://mqtt-explorer.com/). Apart from verifying the messages, you can see all of the messages available from OVMS.

Note that you will need to turn off "Validate Certificate" in MQTT Explorer as the CA you created cannot be truly verified.

## Home Assistant Configuration
### Sensors & YAML
The below assumes that you already have Home Assistant running and a broker configured within Home Assistant.
The sensors are examples of what can be configured but they will need to be modified based on your MQTT topics in use with OVMS which vary on your setup:
```
mqtt:
  binary_sensor:
    - name: "OVMS 12V Battery Alert"
      state_topic: "ovms/CAR/UNIQUEID/metric/v/b/12v/voltage/alert"
      icon: mdi:car-battery
  sensor:
    - name: "OVMS GPS Latitude"
      state_topic: "ovms/CAR/UNIQUEID/metric/v/p/latitude"
      icon: mdi:latitude
    - name: "OVMS GPS Longitude"
      state_topic: "ovms/CAR/UNIQUEID/metric/v/p/longitude"
      icon: mdi:longitude
    - name: "OVMS GPS Signal Strength"
      state_topic: "ovms/CAR/UNIQUEID/metric/v/p/gpssq"
      device_class: signal_strength
      unit_of_measurement: '%'
    - name: "OVMS GPS Time Updated"
      state_topic: "ovms/CAR/UNIQUEID/metric/v/p/gpstime"
      value_template: '{{ value_json | timestamp_local }}'
      device_class: timestamp
    - name: "OVMS 12V Battery"
      state_topic: "ovms/CAR/UNIQUEID/metric/v/b/12v/voltage"
      value_template: '{{ value | round(1) }}'
      icon: mdi:car-battery
      unit_of_measurement: 'V'
    - name: "OVMS 12V Battery Alert"
      state_topic: "ovms/CAR/UNIQUEID/metric/v/b/12v/voltage/alert"
      icon: mdi:car-battery
    - name: "OVMS Battery Charging"
      state_topic: "ovms/CAR/UNIQUEID/metric/v/c/charging"
      icon: mdi:ev-plug-type2
    - name: "OVMS Charging State"
      state_topic: "ovms/CAR/UNIQUEID/metric/v/c/state"
      icon: mdi:battery-sync
    - name: "OVMS Charging Time Until Full"
      state_topic: "ovms/CAR/UNIQUEID/metric/v/c/duration/full"
      device_class: duration
      value_template: '{{ timedelta(minutes=value | int) }}'
      unit_of_measurement: 'hrs'
    - name: "OVMS Charging Current"
      state_topic: "ovms/CAR/UNIQUEID/metric/v/c/current"
      device_class: apparent_power
      unit_of_measurement: 'A'
    - name: "OVMS Charging Voltage"
      state_topic: "ovms/CAR/UNIQUEID/metric/v/c/voltage"
      device_class: voltage
      unit_of_measurement: 'V'
    - name: "OVMS Charging Session"
      state_topic: "ovms/CAR/UNIQUEID/metric/v/c/kwh"
      value_template: '{{ value | round(1) }}'
      unit_of_measurement: 'kW'
      device_class: power
    - name: "OVMS Charging Type"
      state_topic: "ovms/CAR/UNIQUEID/metric/v/c/type"
      icon: mdi:ev-plug-type2
    - name: "OVMS Battery State of Charge"
      state_topic: "ovms/CAR/UNIQUEID/metric/v/b/soc"
      value_template: '{{ value | round(1) }}'
      device_class: battery
      unit_of_measurement: '%'
    - name: "OVMS Battery State of Health"
      state_topic: "ovms/CAR/UNIQUEID/metric/v/b/soh"
      value_template: '{{ value | round(1) }}'
      device_class: battery
      unit_of_measurement: '%'
    - name: "OVMS Ideal Range"
      state_topic: "ovms/CAR/UNIQUEID/metric/v/b/range/ideal"
      value_template: '{{ value | multiply(0.621371) | float | round(0) }}'
      device_class: distance
      unit_of_measurement: 'miles'
    - name: "OVMS Estimated Range"
      state_topic: "ovms/CAR/UNIQUEID/metric/v/b/range/est"
      value_template: '{{ value | multiply(0.621371) | float | round(0) }}'
      device_class: distance
      unit_of_measurement: 'miles'
    - name: "OVMS Odometer"
      state_topic: "ovms/CAR/UNIQUEID/metric/v/p/odometer"
      value_template: '{{ value | multiply(0.621371) | float | round(0) }}'
      device_class: distance
      unit_of_measurement: 'miles'
      icon: mdi:speedometer
    - name: "OVMS 4G Signal Quality"
      state_topic: "ovms/CAR/UNIQUEID/metric/m/net/mdm/sq"
      device_class: signal_strength
    - name: "OVMS 4G Network Reg"
      state_topic: "ovms/CAR/UNIQUEID/metric/m/net/mdm/netreg"
      icon: mdi:radio-tower
```
