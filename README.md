# OMVS using MQTT and Home Assistant
## Overview
This guide shows you link your Open Vehicle Monitoring System (https://www.openvehicles.com/) to use your own MQTT server with TLS encryption, while also reading the data within Home Assistant.

## Preparing MQTT Broker
For the purpose of this section, I will be using a pre-existing Mosquitto MQTT Broker.

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
