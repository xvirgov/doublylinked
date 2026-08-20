---
layout: post
title: "quick notes on the TLS 1.3 handshake"
date: 2026-05-14
tags: [crypto, tls, networking]
---

A short refresher on the TLS 1.3 handshake, written for my future self. This is the simplified happy path — no PSK resumption, no 0-RTT, no HelloRetryRequest.

## the goal

Establish a shared symmetric key between client and server, authenticate the server (and optionally the client), and start sending encrypted application data — in **one round trip**, down from two in TLS 1.2.

## the messages

```
Client                                Server

ClientHello
  + key_share          -------->
                                      ServerHello
                                        + key_share
                                      {EncryptedExtensions}
                                      {Certificate}
                                      {CertificateVerify}
                                      {Finished}
                       <--------      [Application Data]
{Finished}             -------->
[Application Data]     <------->      [Application Data]
```

`{}` = encrypted with handshake keys. `[]` = encrypted with application keys.

## what's actually happening

1. **ClientHello** carries a list of supported ciphersuites *and* an ephemeral (EC)DH public key in `key_share`. The client is guessing what group the server supports — if it guesses wrong, the server sends HelloRetryRequest and we lose the RTT advantage.

2. **ServerHello** picks a ciphersuite and returns its own `key_share`. After this message both sides can derive the **handshake traffic secret** via HKDF over the shared DH output.

3. Everything after ServerHello on the server's side is already encrypted with the handshake key. This is the big shift from 1.2 — certificates are no longer visible to a passive observer.

4. **CertificateVerify** is a signature over the transcript hash, proving the server owns the private key matching the certificate. Critically, this binds the signature to the entire handshake so far, killing a class of cross-protocol attacks.

5. **Finished** is an HMAC over the transcript. Once both sides send it, the handshake is authenticated end-to-end and they switch to application traffic keys.

## why this is a real improvement

- one RTT instead of two
- forward secrecy is mandatory (no static RSA key exchange)
- the cipher list is dramatically smaller, killing most downgrade and misconfiguration footguns
- handshake content (certs, extensions) is encrypted
- a clean key schedule based on HKDF instead of the TLS 1.2 PRF mess

## further reading

- RFC 8446 — the spec, surprisingly readable
- the [miTLS](https://mitls.org/) papers on formal verification of the handshake
- Filippo Valsorda's blog posts on TLS 1.3, all excellent

More on the key schedule and 0-RTT in a future post.
