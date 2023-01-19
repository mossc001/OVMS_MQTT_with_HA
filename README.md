# OMVS using MQTT and Home Assistant
## Overview
This guide shows you link your Open Vehicle Monitoring System (https://www.openvehicles.com/) to use your own MQTT server with TLS encryption, while also reading the data within Home Assistant.

## Preparing MQTT Broker
For the purpose of this section, I will be using a pre-existing Mosquitto MQTT Broker.

### Certificate Generation
#### Certificate Authority [CA]

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
