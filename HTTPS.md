# Base64 Encoding

![Untitled](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/d3a6554e-3fc0-4685-abd3-a6540057c399)

- It is method of converting binary data (such as images, audio files or any other type of file) into ASCII text

![image](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/55ca3f01-da2e-4f25-8559-45e376b89dc4)

- security is vulnerable
- If someone decode the code, they easily get password

<br/>

# Diegest Authentication

- Never send the raw password over network
- Hash the password

![Untitled](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/e59ccb20-a230-4e40-aee2-692d9f139307)

## Hashing

- `Deterministic` : The same input will always result in the `same output`
- `Fixed Size`
- Fast Computation
- Pre-image Resistence : `Infeasible to reverse` the hash back to the original input
- `Collision Resistance` : It should be extremely hard to find two different inputs that produce the same output

![image](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/f1ce0763-d5a5-4df6-908e-e44ce0b47cef)

ex) MD5, SHA-1, SHA-2, SHA-3 etc

- MD5 : It is now considered broken and vulnerable
- SHA-1 : It haas been found vulnerable to collision attacks and is no longer recommended for security-sensitive applications
- SHA-256 and SHA-3 : They are currently considered secure and widely used in various security application and protocols

<br/>

# Nounce

![Untitled](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/b899c25e-0a86-4311-83b7-a54e5976febb)

- Send the nonce which is frequently changed
- Using HTTP digest Authorization Protocol, adding the Authorization-info to share the `nounce`

### Problem
1. Dictionary Attack
2. Man-In-The-Middle Attack : Able to be infected middle of the network

## +) Preemptive authorization

![Untitled](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/ebfa863e-4339-4c7e-a5a9-769d98d8f8a4)

<br/>

# HTTPS

- Created by Netscape Communications Coporation

![Untitled](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/3b81fc64-79a6-4516-8961-f01765bc4768)

- SSL (Secure Socket Layer), TLS (Transport Layer Security)

## SSL Handshacking

![Untitled](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/b983c0dc-ad60-4ee5-afc0-47d331b50159)

- Use `Symmetric Key` And `Asymmetric Key`

## +) SSL VS TLS

### SSL
- Early version of SSL have been deprecated due to various security vulnerabilities that cannot be fixed

### TLS
- The development of TLS was driven by the need for enhanced security, compatibility and extensibility in secure communications over the internet
- Offer stronger security features compared to SSL

### Digital Signature

![image](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/0590af58-a3f6-4660-9cfb-4054f9a33134)

## How to create Site Certification

### 1. Server -> CA (Certificate Authority)

![6afc5c43a5050054d7482202e3b75239](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/dddd053d-cd9b-44d6-9072-ea33fa1db692)

- Create Public Key, Private Key
- Transmit Site information and Public Key (+) CSR)

### 2. CA

![Untitled](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/eb36f3fc-6ceb-4987-b531-ce905dbfb30f)

1. Verify data with Public Key (`Digital Signature`)
2. Issue SSL (CRT) using the information receiving from server
3. For setting SSL authentication, create CA Public Key, Private Key

### 3. CA -> Server

1. Encrypt SSL Certification using Private Key
2. Send SSL Certification and Public Key

### 4. Server

1. Using Public Key, encrypt and verify data
2. If verificaion is success, Issue certification is completed

# How browser verify the CA

![Untitled](https://github.com/JaeYeonLee0621/a-mixed-knowledge/assets/32635539/4c9181a5-1c6f-4bd1-86f0-ab0524a42e97)

1. Check received certification from server is existed in browser's CA certification list
2. If it is existed, encrpyt certification using CA public key which is stored in browser
3. If vertification is completed, using public key to encrypt data

**[Browser trust store updates]**
- Trust Store Updates : Browsers and OS regularly update their trust stores through software updates

+) Trust store : Pre-installed set of trusted CA certificatiions
