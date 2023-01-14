# GnuPG Key Management

GPG is a wonderful tool for encrypting and signing data and authentication.
Using GPG for years, I noticed the onboarding for new users was never easy.

This repository will go through my understanding of GPG [use cases and best practices](#GnuPG-Key-Description), as well as a [technical walkthrough](./Technical-Walkthrough.md)] to get a nice GPG environment set up.

GPG is a powerful tool for encrypting and signing data, as well as for authentication.
Using GPG for years, I've found that the onboarding process for new users is often difficult. This repository aims to share my understanding of GPG  [use cases and best practices](#GnuPG-Key-Description), as well as provide a  [technical walkthrough](./Technical-Walkthrough.md)] for setting up a safe GPG environment easily.

---

-  **[GnuPG Key Description](#GnuPG-Key-Description)**
  - **[Different Uses of GPG Keys](./README.md#Different Uses of GPG Keys)**
  - **[Best Practices for Individual GPG Key Management](./README.md#Best Practices for Individual GPG Key Management)**
  - **[GPG Key Safety on a SmartCard](./README.md#GPG Key Safety on a SmartCard)**
-  **[GnuPG Key Technical Walkthrough](./Technical-Walkthrough.md)**
  - **[Create gpg key pair](./Technical-Walkthrough.md#Create gpg key pair)**
  - **[Create Subkeys](./Technical-Walkthrough.md#Create Subkeys)**
  - **[Key Rotation? Sign this new key ](./Technical-Walkthrough.md#Key Rotation? Sign this new key )**
  - **[Protect your master key](./Technical-Walkthrough.md#Protect your master key)**
  - **[Use the offline master key](./Technical-Walkthrough.md#Use the offline master key)**
  - **[Protect your subkeys](./Technical-Walkthrough.md#Protect your subkeys)**
  - **[Test your key](./Technical-Walkthrough.md#Test your key)**
  - **[Import /  Re-import](./Technical-Walkthrough.md#Import /  Re-import)**
  - **[Deletion / Revocation](./Technical-Walkthrough.md#Deletion / Revocation)**
  - **[References](./Technical-Walkthrough.md#References)**

---

## GnuPG Key Description

### Different Uses of GPG Keys

GPG, or GNU Privacy Guard, is a free and open-source implementation of the OpenPGP standard for encrypting and signing data. GPG keys are used for a variety of purposes, including:

- Email encryption: GPG can be used to encrypt and sign email messages to ensure that they can only be read by the intended recipient.
- File encryption: GPG can be used to encrypt files to protect them from unauthorized access.
- Code signing: Developers can use GPG to sign their code to verify that it has not been tampered with and to establish the identity of the developer.
- SSH authentication: GPG keys can be used as an alternative to traditional SSH keys for logging into remote servers.
- Digital signatures: GPG keys can be used to create digital signatures, which can be used to verify the authenticity of documents or software.

Overall, GPG provides a powerful way to secure communication, protect data, and establish the identity of the sender.

### Best Practices for Individual GPG Key Management

- Use a strong passphrase to protect your private key.
- Keep a backup of your private key in a secure location.
- Regularly update your key, including extending its expiration date.
- Share your public key widely to allow others to send you encrypted messages and files.
- Sign other people's keys to establish trust in the web of trust.
- Revoke your key if it becomes compromised or you no longer use it.
- Use a key management software to easily manage your keys and settings.
- Use a hardware security device, such as a YubiKey, to store your private key and add an extra layer of security.
- Be aware of phishing attempts and never share your private key with anyone.

### GPG Key Safety on a SmartCard

#### TL;DR

Using a SmartCard to store GPG keys can increase security by adding an additional layer of protection to the keys. The SmartCard can be used to encrypt and decrypt the keys, and can also be used as a hardware token for authenticating to GPG. Additionally, storing the keys on the SmartCard allows them to be easily transported and used on different computers without the need to copy the keys to each machine. However, it is important to properly secure the SmartCard to prevent unauthorized access to the keys.

#### Advantages of Moving GPG Keys to a OpenPGP SmartCard (YubiKey or Alternative)

YubiKey is mainly known for its two-factor authentication, but its GPG smart-card functionality makes it very useful for GPG key management.

There are several advantages to moving GPG keys to an OpenPGP SmartCard:

- Physical security: Storing GPG keys on a SmartCard provides an additional layer of security by keeping the keys on a physical device that can be locked up and protected from unauthorized access.
- Portability: With the keys stored on a SmartCard, you can use them on multiple devices without having to transfer them between computers.
- Convenience: Using a SmartCard for GPG key management makes it easier to perform tasks such as signing and encrypting messages, as the keys are always available on the device.
- Tamper-proof: YubiKey is tamper-proof, which makes it difficult for an attacker to extract the keys or modify the key material. Furthermore, YubiKey hardware can encrypt and decrypt the key material, which makes it more secure than storing the key on disk.

It is generally considered difficult to extract a GPG key from a YubiKey, as the device is designed to be tamper-resistant and secure. The keys are stored in a secure element inside the YubiKey, which is protected by several layers of hardware and software security. The keys are encrypted and protected by a PIN, and the secure element includes a built-in tamper-detection mechanism that will wipe the keys if an unauthorized attempt is made to extract them.

The keys are encrypted and protected by a PIN, and the secure element includes a built-in tamper-detection mechanism that will wipe the keys if an unauthorized attempt is made to extract them.

However, it is important to note that no security measure is completely foolproof, and that any device can potentially be hacked or compromised if a hacker has enough resources and expertise. Additionally, if the SmartCard is lost or stolen, it could be used by an attacker with physical access to the device.

It is always a good practice to keep a backup of the GPG key and use a strong passphrase to protect the key from brute-force attacks.

---

[Gwenall Pansier](https://github.com/gwenall) - [gwenall.pansier+git@my-cloud.me](mailto:gwenall.pansier+git@my-cloud.me)

