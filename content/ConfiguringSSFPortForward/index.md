---
title: "Configuring SSF to Port Forward"
subtitle: "Port Forwarding without Access to a Router"
date: 2020-03-06T12:57:17-04:00
lastmod: 2020-05-29T12:57:17-04:00
draft: false
author: "Nathan Higley"
authorLink: ""
description: ""

tags: [networking, SSF, port forward]
categories: [Guides, Networking, Security]

hiddenFromHomePage: false
hiddenFromSearch: false

featuredImage: ""
featuredImagePreview: ""

toc:
  enable: true
math:
  enable: false
lightgallery: false
license: ""
---

# Configuring SSF for Port Forwarding


To configure SSF you have to do different things on the server with the public facing IP and on the client which runs the service you want to forward.

### Install SSF

#### Do on both Server and Client
Download SSF: [SSF - Secure Socket Funneling - Network tool - TCP and UDP port forwarding, SOCKS proxy, Remote shell, Native Relay protocol, Standalone](https://securesocketfunneling.github.io/ssf/#security-features)

Extract to /opt/ssf

>\# unzip *.zip /opt/ssf

### Setup keys
> [SSF - Secure Socket Funneling - Network tool - TCP and UDP port forwarding, SOCKS proxy, Remote shell, Native Relay protocol, Standalone](https://securesocketfunneling.github.io/ssf/#security-features) 

#### Do on both the Server and Client
Generate Diffie-Hellman parameters
>\# openssl dhparam 4096 -outform PEM -out dh4096.pem
#### Do on the Client
Generate a Certificate Authority
>\# openssl req -x509 -nodes -newkey rsa:4096 -keyout ca.key -out ca.crt -days 3650
#### Copy to Server
>\# scp  /opt/ssf/certs/ca.*  user@\<server-ip>:/opt/ssf/certs/
#### Do on both the Server and Client
Create 'extfile.txt':
> \# touch extfile.txt
```
[ v3_req_p ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment

[ v3_ca_p ]
basicConstraints = CA:TRUE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment, keyCertSign
```

Generate private key and certificate (leave fields blank):
> \# openssl req -newkey rsa:4096 -nodes -keyout private.key -out certificate.csr

> \# openssl x509 -extfile extfile.txt -extensions v3_req_p -req -sha1 -days 3650 -CA ca.crt -CAkey ca.key -CAcreateserial -in certificate.csr -out certificate.crt

> \# cat ca.crt >> certificate.crt

Encrypt private key with password:
> \# openssl rsa -in private.key -out private.key -aes256 -passout pass:\<password>

Move ca.crt to Trusted Folder:
> \# mv /opt/ssf/certs/ca.crt /opt/ssf/certs/trusted/ca.crt

### Create Configuration File for SSF

#### On Server

>\# touch /opt/ssf/ssf.conf

```
{
  "ssf": {
    "arguments": "",
    "circuit": [],
    "tls" : {
      "ca_cert_path": "/opt/ssf/certs/trusted/ca.crt",
      "cert_path": "/opt/ssf/certs/certificate.crt",
      "key_path": "/opt/ssf/certs/private.key",
      "key_password": "<server-private-key-password>",
      "dh_path": "/opt/ssf/certs/dh4096.pem",
      "cipher_alg": "DHE-RSA-AES256-GCM-SHA384"
    },
    "http_proxy" : {
      "host": "",
      "port": "",
      "user_agent": "",
      "credentials": {
        "username": "",
        "password": "",
        "domain": "",
        "reuse_ntlm": "true",
        "reuse_nego": "true"
      }
    },
    "services": {
      "datagram_forwarder": { "enable": false },
      "datagram_listener": {
        "enable": false,
        "gateway_ports": false
      },
      "stream_forwarder": { "enable": false },
      "stream_listener": {
        "enable": true,
        "gateway_ports": true
      },
      "copy": { "enable": false },
      "shell": {
        "enable": false,
        "path": "/bin/bash|C:\\windows\\system32\\cmd.exe",
        "args": ""
      },
      "socks": { "enable": false }
    }
  }
}

```

#### On Client

>\# touch /opt/ssf/ssf.conf

```
{
  "ssf": {
    "arguments": "",
    "circuit": [],
    "tls" : {
      "ca_cert_path": "/opt/ssf/certs/trusted/ca.crt",
      "cert_path": "/opt/ssf/certs/certificate.crt",
      "key_path": "/opt/ssf/certs/private.key",
      "key_password": "<client-private-key-password>",
      "dh_path": "/opt/ssf/certs/dh4096.pem",
      "cipher_alg": "DHE-RSA-AES256-GCM-SHA384"
    },
    "http_proxy" : {
      "host": "",
      "port": "",
      "user_agent": "",
      "credentials": {
        "username": "",
        "password": "",
        "domain": "",
        "reuse_ntlm": "true",
        "reuse_nego": "true"
      }
    },
    "services": {
      "datagram_forwarder": { "enable": false },
      "datagram_listener": {
        "enable": false,
        "gateway_ports": false
      },
      "stream_forwarder": { "enable": true },
      "stream_listener": {
        "enable": false,
        "gateway_ports": true
      },
      "copy": { "enable": false },
      "shell": {
        "enable": false,
        "path": "/bin/bash|C:\\windows\\system32\\cmd.exe",
        "args": ""
      },
      "socks": { "enable": false }
    }
  }
}

```

### Configure Systemd Services

> \# /etc/systemd/system/ssf-server.service
```
[Unit]
Description=SSF Server Service
After=network.target

[Service]
User=root
ExecStart=/opt/ssf/ssfd -p <port-to-host-ssf-on> -c /opt/ssf/ssf.conf -g

[Install]
WantedBy=multi-user.target

```

> \# /etc/systemd/system/ssf-client.service
```
[Unit]
Description=SSF Client Service
After=network.target

[Service]
User=root
ExecStart=/opt/ssf/ssf -R 0.0.0.0:<port-to-forward>:127.0.0.1:<port-to-forward> -p <port-to-host-ssf-on> <server-public-ip> -c /opt/ssf/ssf.conf -g

[Install]
WantedBy=multi-user.target
```

### Enable Services and Start Them

#### On Server

> \# systemctl enable ssf-server.service

> \# service ssf-server start

#### On Server

> \# systemctl enable ssf-client.service

> \# service ssf-client start


