---
title: "Network: TLS"
date: "2023-01-03"
weight: 90
---

**TL;DR** TLS (Transport Layer Security) is a network security protocol that uses cryptographic algorithms involving pairs of keys (private and public) to secure network communications.

---

When you talk to google.com, how do you know the response you receive is sent from a legit server?

The server hands you a **certificate** for the domain. With the certificate, it claims to be the owner of the domain. On the certificate, you can see a domain name, a public key, and a digital signature.

### **Chain of certificates**

First, how do you know the certificate is legit? The digital signature indicates that this certificate is validated by another entity. You also get the certificate of that signing entity. That certificate will have another signature given by another entity. If you follow the chain of certificates you will reach the **root certificate** which is self-signed.

How do you know the root certificate is valid? Your browser or operating system has been **pre-installed** with a list of valid root certificates. You'll have to *assume* those are legit. Of course, it's possible that they are compromised during the installation process but that should be very rare.

### Certificate Authority (CA)

A **certificate authority** (CA) issues certificates to other entities.

Each certificate has a public key on it. The entity that has the corresponding private key **owns** the certificate.

A **root CA** issues the **root certificate** with the following process

1. It generates a pair of public and private keys.
2. It puts the public key on a certificate.
3. It uses the private key to sign the certificate. This certificate is self-signed.

After that, the root CA can sign certificates issued to **intermediate CAs**.

1. An intermediate CA first generates a pair of public and private keys.
2. It sends its public key to the root CA and requests a certificate.
3. The root CA generates a certificate with the requested public key on it, signs it with its own private key, and sends it to the intermediate CAs. The signature also points to the signing authority, which is the root CA in this case.

After this, the intermediate CAs can sign certificates issued to end users.

How does a CA know whether it should issue a certificate? A CA needs to independently verify this. For a domain-validated certificate, the CA sends the certificate to the entity that's listed as the domain owner in the WHOIS record. The security level is pretty low though because it's easy for malicious users to set up a phishing website with misspellings and obtain such certificates for it. They will still see HTTPS is used but may not notice the misspellings.

### Authentication

OK, now we know the certificate is valid, how to verify that the server really owns what it claims to own?

The owner of the certificate must know the **private key** corresponding to the **public key** on the certificate.

You use the public key on the certificate to encrypt some **random information**. Only the entity with its private key can decrypt it and **tell you what that random information is**. This way you are sure that you are really talking to the owner.

**Asymmetric Encryption** is usually expensive. After the initial exchanges, the two parties will switch to **Symmetric Encryption** where they use the same key for encryption and decryption.

### Concept Clarification

TLS is the latest security protocol. An older version was called **SSL**.

TLS is based on the RSA. **RSA** is a public-key **encryption mechanism**.

### The Magic of RSA

Without digging into too much math for now, here are what properties a pair of private/public keys has

- A piece of information encrypted by the **public key** can only be decrypted by the **private key**
- A piece of information encrypted by the **private key** can only be decrypted by the **public key**

**Digital Signature**

- A signature of a document is obtained by calculating the hash of the document and encrypting that hash with the private key.
- Anyone with the public key can decrypt the value and compare it with the hash of the document. If they match, then
  - The signature comes from an entity owning the private key
  - That entity endorses the content of the document.

RSA can be used for

- Identification.
- Encryption.
- Key Exchange.

It involves 2048-bit semi-prime numbers.
