---
layout: post
title: "SSL certificate setup"
author: "Shtille"
categories: journal
tags: [thoughts,bash,SSL]
---

Once I decided to make my site available through HTTPS protocol, I needed a SSL sertificate.
There two solutions to obtain one:
- Make self-signed certificate
- Buy certificate from trusted CA

## Make self-signed certificate

Self-signed certificate can be generated with one command:
```bash
openssl req -x509 -newkey rsa:4096 -sha256 -days 3650 -nodes -keyout key.pem -out cert.pem -subj "/CN=shtille.space" -addext "subjectAltName=DNS:shtille.space,DNS:*.shtille.space,IP:91.107.126.200"
```
But it doesn't work well on browsers since it's not from known certificate authority.
We may use it for _localhost_ just for test purposes.

## Buy certificate from trusted CA

I bought one from GlobalSign. And I received four files:
- GlobalSign Root Certificate
- GlobalSign Intermediate Certificate
- My GlobalSign SSL Certificate (in PEM format)
- My GlobalSign SSL Certificate (in P7B format)

A private key is downloaded separately from vendor's site.

### How to make certificate chain file
```bash
cat GlobalSign\ Root\ CA.crt >> GlobalSign.ca
cat AlphaSSL\ CA\ -\ SHA256\ -\ G4.crt >> GlobalSign.ca
```

### How to run server in HTTPS mode
```bash
./web-server/build/web-server --port 443 --site ./shtille.space/ --blog ./blog/ \
	--key ../SSL/shtille_space.key --cert ../SSL/shtille_space.crt --trust ../SSL/GlobalSign.ca
```