---
title: "Network-Infrastructure Hiding Protocol"
abbrev: "NHP"
category: info
docname: draft-opennhp-ztcpp-nhp-latest
submissiontype: independent
v: 3
keyword:
 - zero trust
 - session layer
 - network obfuscation
 - SDP
venue:
  group: "ztcpp"
  type: "Independent Submission"
  mail: "ztcpp@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/ztcpp/"
  github: "OpenNHP/ietf-rfc-nhp"
  latest: "https://OpenNHP.github.io/ietf-rfc-nhp/draft-opennhp-ztcpp-nhp.html"

author:
 -
    fullname: Benfeng Chen
    organization: OpenNHP
    email: benfeng@gmail.com
 -
    fullname: Justin Posey
    organization: LayerV

normative:
  RFC2119:
  RFC8174:
  RFC9000:
  RFC8446:
  RFC9180:
  NoiseFramework:
    title: "The Noise Protocol Framework"
    author:
      name: Trevor Perrin
    date: 2018
    target: https://noiseprotocol.org/noise.html

informative:
  RFC8126:
  NIST.SP.800-207:
    title: "Zero Trust Architecture"
    author:
      - name: Scott Rose
      - name: Oliver Borchert
      - name: Stu Mitchell
      - name: Sean Connelly
    date: 2020
    seriesinfo:
      NIST: Special Publication 800-207
  CSA.SDP.Spec2.0:
    title: "Software Defined Perimeter Specification v2.0"
    author:
      org: Cloud Security Alliance
    date: 2022
  CSA.NHP.Whitepaper:
    title: "Stealth Mode SDP for Zero Trust Network Infrastructure: Introducing the Network-Infrastructure Hiding Protocol (NHP)"
    author:
      org: Cloud Security Alliance
    date: 2026

--- abstract

The Network-Infrastructure Hiding Protocol (NHP) is a cryptography-based session-layer protocol designed to operationalize Zero Trust principles by concealing protected network resources from unauthorized entities. NHP enforces authentication-before-connect access control, rendering IP addresses, ports, and domain names invisible to unauthorized users. This document defines the protocol architecture, cryptographic framework, message formats, and workflow to enable independent implementation of NHP. It represents the third generation of network hiding technology—evolving from first-generation port knocking to second-generation Single-Packet Authorization (SPA) and now to NHP with advanced asymmetric cryptography, mutual authentication, and scalability for modern threats. This specification also provides guidance for integration with Software-Defined Perimeter (SDP), DNS, FIDO, and Zero Trust policy engines.

--- middle

# Introduction

Since its inception in the 1970s, the TCP/IP networking model has prioritized openness and interoperability, laying the foundation for the modern Internet. However, this design philosophy also exposes systems to reconnaissance and attack. As Vint Cerf, who personally designed many of these components, stated, "We didn't focus on how you could wreck this system intentionally."

Today, the cyber threat landscape has been dramatically reshaped by the rise of AI-driven attacks, which bring unprecedented speed and scale to vulnerability discovery and exploitation. Automated tools continuously scan the global network space, identifying weaknesses in real-time. Large Language Models (LLMs) can now autonomously exploit one-day vulnerabilities, and AI systems can generate working exploits for published CVEs in minutes. As a result, the Internet is evolving into a "Dark Forest," where **visibility equates to vulnerability**. In such an environment, any exposed service becomes an immediate target.

The Zero Trust model, which mandates continuous verification and eliminates implicit trust, has emerged as a modern approach to cybersecurity. Within this context, the Network-Infrastructure Hiding Protocol (NHP) offers a new architectural element: authenticated-before-connect access at the session layer.

NHP builds upon foundational work in the Cloud Security Alliance's Software-Defined Perimeter (SDP) and Single-Packet Authorization (SPA) frameworks, representing the third generation of network hiding technology:

* **First Generation - Port Knocking:** Simple port sequences vulnerable to interception and replay attacks.
* **Second Generation - SPA:** Encrypted single-packet authorization with improved security but limited scalability.
* **Third Generation - NHP:** Advanced asymmetric cryptography, mutual authentication, Noise Protocol-based key exchange, and enterprise-grade scalability.

This document outlines the motivations behind NHP, its design objectives, message structures, integration options, and security considerations for adoption within Zero Trust frameworks.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

The following terms are used throughout this document:

NHP
: Network-Infrastructure Hiding Protocol

NHP-Agent
: The client-side component that initiates NHP communication

NHP-Server
: The control-plane service that validates requests and makes access decisions

NHP-AC
: NHP Access Controller, the enforcement component near protected resources

SPA
: Single-Packet Authorization

SDP
: Software-Defined Perimeter

ZTA
: Zero Trust Architecture

ECC
: Elliptic Curve Cryptography

AEAD
: Authenticated Encryption with Associated Data

ASP
: Authorization Service Provider

PEP
: Policy Enforcement Point

KGC
: Key Generation Center

# Design Objectives

The NHP protocol is designed to achieve the following objectives:

1. **Infrastructure Invisibility:** Eliminate unauthorized network visibility by enforcing authentication prior to session establishment. Protected resources remain invisible to unauthorized scanners and attackers.

2. **Session Layer Operation:** Operate at OSI Layer 5, complementing existing TCP, UDP, and QUIC transports without requiring changes to underlying network infrastructure.

3. **Decentralized Trust:** Support decentralized trust using asymmetric cryptography and ephemeral key exchange, eliminating single points of trust failure.

4. **Fine-Grained Access Control:** Enable context-based policy enforcement across heterogeneous environments, supporting least-privilege access.

5. **Integration Capability:** Integrate with existing Zero Trust controllers, SDP gateways, identity systems (IAM), DNS infrastructure, and FIDO authentication.

6. **Scalability:** Support enterprise-scale deployments with clustered servers, distributed access controllers, and multi-tenant isolation.

7. **AI Threat Mitigation:** Reduce the attack surface against AI-driven reconnaissance and exploitation by denying visibility before authentication.

# Relationship to TLS

NHP and TLS (Transport Layer Security) are complementary protocols that operate at different OSI layers and serve distinct security purposes. This section clarifies their differences and how they work together.

## OSI Layer Positioning

~~~
+-------------------+
| Application (L7)  |  HTTP, SMTP, SSH, etc.
+-------------------+
        ↓
+-------------------+
| Presentation (L6) |  TLS/SSL - Data encryption & integrity
+-------------------+
        ↓
+-------------------+
| Session (L5)      |  NHP - Authentication before connection
+-------------------+
        ↓
+-------------------+
| Transport (L4)    |  TCP, UDP, QUIC
+-------------------+
        ↓
+-------------------+
| Network (L3)      |  IP
+-------------------+
~~~

## Key Differences

| Aspect | NHP (Layer 5) | TLS (Layer 6) |
|--------|---------------|---------------|
| **Primary Purpose** | Infrastructure hiding and access control | Data encryption and integrity |
| **When Authentication Occurs** | BEFORE connection establishment | AFTER TCP connection established |
| **Service Visibility** | Services are INVISIBLE to unauthorized users | Services are VISIBLE, communication is encrypted |
| **Attack Surface** | Eliminates pre-authentication attack surface | Protects data in transit, but service ports remain exposed |
| **Port Exposure** | No ports exposed until authenticated | Ports must be open to initiate TLS handshake |
| **Vulnerability Window** | None—no connection without authentication | TLS handshake vulnerabilities can be exploited |

## The Pre-Authentication Problem

TLS provides excellent protection for data in transit, but it has a fundamental limitation: **the service must be reachable to initiate the TLS handshake**. This creates a pre-authentication attack window:

~~~
Traditional TLS Flow:

Attacker    ──────►  Open Port 443  ──────►  TLS Handshake  ──────►  Authentication
                         ↑
                    Service is VISIBLE
                    Port scan succeeds
                    Pre-auth exploits possible
~~~

~~~
NHP + TLS Flow:

Attacker    ──────►  No Open Ports  ──────►  BLOCKED (Service Invisible)
                         ↑
                    Cannot discover service
                    Port scan fails

Authorized  ──────►  NHP Knock  ──────►  Port Opens  ──────►  TLS  ──────►  Application
User                     ↑                    ↑
                    Authenticated         Encrypted
                    BEFORE connect        data transfer
~~~

## Complementary Security Model

NHP and TLS are designed to work together, not replace each other:

1. **NHP provides:** Authentication-before-connect, infrastructure invisibility, access control
2. **TLS provides:** Data encryption, integrity verification, server authentication

A complete Zero Trust deployment SHOULD use both:

* **NHP** ensures only authorized users can discover and reach the service
* **TLS** encrypts all data exchanged after access is granted

## Vulnerabilities Addressed by NHP but Not TLS

| Vulnerability Type | TLS Protection | NHP Protection |
|--------------------|----------------|----------------|
| Port scanning and service discovery | ✗ None | ✓ Service invisible |
| Pre-authentication exploits (e.g., Heartbleed) | ✗ Vulnerable | ✓ No connection possible |
| TLS implementation bugs before handshake | ✗ Vulnerable | ✓ No handshake initiated |
| DDoS attacks on exposed services | ✗ Service reachable | ✓ Service hidden |
| Credential stuffing on login pages | ✗ Page accessible | ✓ Page invisible |
| Zero-day exploits before authentication | ✗ Service exposed | ✓ Service protected |

## Why Both Are Needed

NHP alone does not encrypt application data—it only controls access. TLS alone does not hide services—it only encrypts traffic. Together, they provide defense in depth:

* **Without NHP:** Attackers can scan, probe, and exploit services before any authentication occurs
* **Without TLS:** Authorized traffic would be transmitted in plaintext after NHP grants access
* **With Both:** Services are invisible to attackers, and all authorized traffic is encrypted

This layered approach aligns with Zero Trust principles: never trust, always verify, and minimize attack surface at every layer.

# Threat Model

NHP is designed to mitigate the following threat categories:

## Reconnaissance and Scanning

Automated scanning tools and AI-driven reconnaissance continuously probe Internet-facing services. NHP eliminates the ability to discover protected resources by requiring cryptographic authentication before any network visibility is granted.

## Pre-Authentication Exploits

Many vulnerabilities can be exploited before authentication occurs. By enforcing authentication-before-connect, NHP prevents attackers from reaching vulnerable services.

## DDoS Attacks

NHP reduces DDoS attack surface by hiding service endpoints. Attackers cannot target what they cannot discover.

## Credential Theft and Replay

NHP uses ephemeral keys and timestamp-based nonces to prevent credential replay attacks. Each session requires fresh cryptographic material.

## Man-in-the-Middle Attacks

Mutual authentication using asymmetric cryptography ensures both parties verify each other's identity before establishing communication.

# Architectural Overview

NHP operates as a distributed session-layer protocol that enforces authentication-before-connect access between clients and protected resources.

## Core Components

### NHP-Agent

The NHP-Agent is a client-side process, SDK, or embedded module that initiates communication with the protected network. Its responsibilities include:

* Generating and sending NHP-KNK (Knock) messages to the NHP-Server
* Performing cryptographic key exchange using Noise Protocol handshakes
* Managing client identity credentials and device attestation
* Handling session lifecycle including keepalives and re-authentication

### NHP-Server

The NHP-Server is the core control-plane service responsible for:

* Receiving and validating NHP-KNK messages from NHP-Agents
* Authenticating the NHP-Agent identity and device posture
* Interfacing with external Authorization Service Providers (ASP) or IAM systems
* Evaluating access policies based on identity, context, and resource attributes
* Instructing NHP-AC components to open or close access paths
* Managing session state and expiration

Functionally, the NHP-Server maps to the **Policy Administrator** role defined in NIST SP 800-207 Zero Trust Architecture.

### NHP-AC (Access Controller)

The NHP-AC is the enforcement component residing logically or physically near protected resources. Its responsibilities include:

* Maintaining default-deny firewall rules for all protected resources
* Receiving NHP-AOP (AC Operations) commands from the NHP-Server
* Temporarily opening access paths for authorized NHP-Agents
* Automatically reverting to default-deny state when sessions expire
* Reporting access logs and status to the NHP-Server

The NHP-AC corresponds to the **Policy Enforcement Point (PEP)** in NIST SP 800-207 terminology.

### Authorization Service Provider (ASP)

The ASP is an external identity and policy service that the NHP-Server queries for authorization decisions. This may include:

* Identity Providers (IdP) such as LDAP, Active Directory, or OIDC providers
* Policy Decision Points (PDP) implementing ABAC or RBAC policies
* Device posture assessment services
* Risk scoring engines

## Component Interactions

The following diagram illustrates the relationship between NHP components:

~~~
+-------------+          +-------------+          +-------------+
|             |  NHP-KNK |             |  Auth    |             |
| NHP-Agent   |--------->| NHP-Server  |<-------->|    ASP      |
|             |<---------|             |  Query   |   (IAM)     |
+-------------+  NHP-ACK +-------------+          +-------------+
      |                        |
      |                        | NHP-AOP
      |                        v
      |                  +-------------+
      |    NHP-ACC       |             |
      +----------------->|   NHP-AC    |
      |                  |             |
      v                  +-------------+
+-------------+                |
|  Protected  |<---------------+
|  Resource   |   Data Plane
+-------------+
~~~

## Deployment Models

NHP components can be deployed in different configurations:

### Standalone Deployment

For small environments or testing scenarios, the NHP-Server and NHP-AC can coexist on the same host. This configuration simplifies setup while maintaining full protocol compliance.

### Clustered Deployment

In enterprise or cloud environments, multiple NHP-Servers can be deployed in a load-balanced cluster. Each server manages a pool of NHP-AC instances distributed across data centers or network segments. The NHP-Agent dynamically discovers the nearest NHP-Server through DNS or bootstrap configuration.

### Edge AC Deployment

Edge nodes (e.g., gateways, routers, or micro-segmentation agents) can host lightweight NHP-AC components. These edge ACs enforce fine-grained policies close to workloads, improving latency and fault isolation.

### Multi-Tenant Deployment

In service-provider or multi-cloud environments, each tenant can operate an independent NHP-Server while sharing an underlying AC infrastructure. The NHP protocol's namespace isolation ensures complete tenant separation through identity-scoped keys and per-tenant policy databases.

# Protocol Workflow

## Control Plane vs Data Plane

The **Control Plane** carries cryptographic authentication and authorization information among NHP-Agent, NHP-Server, NHP-AC, and optional external ASP. Control plane messages are encrypted using Noise Protocol handshakes.

The **Data Plane** carries application data between the resource requester (NHP-Agent host) and the protected resource, but only after NHP-AC explicitly authorizes access.

This strict separation enforces the *authenticate-before-connect* principle central to Zero Trust.

## Workflow Steps

The complete NHP workflow consists of the following steps:

1. **Knock Request:** NHP-Agent sends NHP-KNK message to NHP-Server containing encrypted identity claims and access request.

2. **Authorization Query:** NHP-Server validates the cryptographic envelope and queries ASP for authorization decision.

3. **Authorization Response:** ASP returns authorization decision with granted permissions and session parameters.

4. **Door Opening:** NHP-Server sends NHP-AOP command to NHP-AC instructing it to open access for the specific NHP-Agent.

5. **AC Confirmation:** NHP-AC enforces the access rule and replies with NHP-ART confirming the operation.

6. **Agent Notification:** NHP-Server sends NHP-ACK to NHP-Agent with access token and connection parameters.

7. **Resource Access:** NHP-Agent sends NHP-ACC to NHP-AC and establishes data plane connection to protected resource.

8. **Session Maintenance:** NHP-Server and NHP-AC maintain session state through NHP-KPL keepalive messages.

9. **Logging and Audit:** Logging is described in {{logging-transmission}}. Log transport is not defined in this revision.

## Sequence Diagram

~~~
NHP-Agent           NHP-Server            NHP-AC             ASP/IAM
    |                    |                    |                   |
    |--- NHP-KNK ------->|                    |                   |
    |                    |--- Auth Query -----|------------------>|
    |                    |<-- Auth Result ----|-------------------|
    |                    |                    |                   |
    |                    |--- NHP-AOP ------->|                   |
    |                    |<-- NHP-ART --------|                   |
    |                    |                    |                   |
    |<-- NHP-ACK --------|                    |                   |
    |                    |                    |                   |
    |--- NHP-ACC --------|------------------>|                   |
    |<================== Data Session ======>|                   |
    |                    |                    |                   |
~~~

# Cryptographic Framework

NHP employs the Noise Protocol Framework {{NoiseFramework}} for all cryptographic operations. This section defines the required cryptographic primitives and handshake patterns.

## Cryptographic Primitives

Implementations MUST support the following cryptographic primitives:

| Function | Algorithm | Reference |
|----------|-----------|-----------|
| DH | Curve25519 | RFC 7748 |
| Cipher | ChaCha20-Poly1305 | RFC 8439 |
| Hash | SHA-256 | RFC 6234 |
| Key Derivation | HKDF | RFC 5869 |

Implementations MAY additionally support:

| Function | Algorithm | Reference |
|----------|-----------|-----------|
| DH | P-256 (secp256r1) | RFC 8422 |
| Cipher | AES-256-GCM | RFC 5116 |
| Hash | BLAKE2s | RFC 7693 |

## Noise Protocol Handshake Patterns

NHP supports the following Noise handshake patterns:

### XX Pattern (Default)

The XX pattern provides full forward secrecy and identity protection for both parties. It is the RECOMMENDED pattern for most deployments.

~~~
XX:
  -> e
  <- e, ee, s, es
  -> s, se
~~~

### IK Pattern (Performance Optimized)

The IK pattern is used when the NHP-Agent knows the NHP-Server's static public key in advance, reducing round trips.

~~~
IK:
  <- s
  ...
  -> e, es, s, ss
  <- e, ee, se
~~~

### K Pattern (One-Way)

The K pattern is used for one-way initiation where only the initiator needs to be authenticated by the responder.

~~~
K:
  <- s
  ...
  -> e, es, ss
~~~

## Key Management

### Static Keys

Each NHP component maintains a static Curve25519 key pair:

* NHP-Agent: Used for client identity and authentication
* NHP-Server: Used for server identity and authentication
* NHP-AC: Used for secure communication with NHP-Server

Static public keys MUST be distributed through a secure out-of-band mechanism or registered through the NHP-REG message flow.

### Ephemeral Keys

Ephemeral keys are generated for each session to provide forward secrecy. Implementations MUST use cryptographically secure random number generators for ephemeral key generation.

### Key Rotation

Static keys SHOULD be rotated periodically. The NHP-REG and NHP-RAK messages support key re-registration without service interruption.

# Message Format

All NHP messages share a common header structure followed by an encrypted payload.

## Message Header

Every NHP message begins with a fixed-length header. The header length depends on the cipher scheme. The reference implementation defines two layouts:

| Layout | Flags bit 0 | Header size | Public key size |
|--------|-------------|-------------|-----------------|
| Curve (CIPHER_SCHEME_CURVE) | 0 | 240 bytes | 32 bytes |
| GMSM (CIPHER_SCHEME_GMSM) | 1 | 304 bytes | 64 bytes |

The header is followed by the encrypted body. All integers are in network byte order.

### Curve Header Layout

| Offset | Size | Field |
|--------|------|-------|
| 0 | 4 | Preamble: random 32-bit value, chosen per message |
| 4 | 4 | Type and Payload Size, XORed with the Preamble |
| 8 | 1 | Major Version |
| 9 | 1 | Minor Version |
| 10 | 2 | Flags |
| 12 | 4 | Unused (not written by the reference implementation) |
| 16 | 8 | Counter |
| 24 | 32 | Ephemeral Public Key |
| 56 | 80 | Identity: 64 bytes of encrypted identity plus a 16-byte AEAD tag |
| 136 | 48 | Static Public Key: 32 bytes plus a 16-byte AEAD tag |
| 184 | 24 | Timestamp: 8 bytes plus a 16-byte AEAD tag |
| 208 | 32 | HMAC |

The GMSM layout has the same field order. The Ephemeral Public Key is 64 bytes and the Static Public Key field is 80 bytes (64 bytes plus a 16-byte tag), which brings the header to 304 bytes.

### Header Fields

Preamble (32 bits)
: A random value chosen per message. The Type and Payload Size field is masked with it, so the type and size are not visible on the wire in plaintext.

Type and Payload Size (32 bits)
: The message type in the upper 16 bits and the length of the encrypted body in the lower 16 bits, XORed with the Preamble. The receiver recovers both by XORing the field with the Preamble. See {{message-types}} for type values.

Version (16 bits)
: Major and minor protocol version. Each is one byte.

Flags (16 bits)
: Bits are numbered from the least significant bit.
  * Bit 0: Extended header. When set, the header uses the GMSM layout.
  * Bit 1: Body compression enabled.
  * Bit 2: Client public key flag.
  * Bits 3-11: Reserved. Senders MUST set these to zero.
  * Bits 12-15: Cipher scheme (0 = Curve, 1 = GMSM).

Counter (64 bits)
: A per-session message counter in network byte order. It is used as the low 8 bytes of the 12-byte AEAD nonce, and the high 4 bytes of the nonce are zero. A counter value MUST NOT be reused within a session with the same key.

Ephemeral, Identity, Static, Timestamp, HMAC
: Key and authentication fields for the handshake. Each encrypted field carries its own AEAD tag. The HMAC covers the header prefix.

The encrypted body follows the header. Its length is given by the Payload Size field. The body is encrypted with the chain hash as additional authenticated data.

## Message Types {#message-types}

| Type Code | Name | Direction | Description |
|-----------|------|-----------|-------------|
| 0x00 | NHP-KPL | Any | Keepalive |
| 0x01 | NHP-KNK | Agent→Server | Knock request |
| 0x02 | NHP-ACK | Server→Agent | Knock acknowledgment |
| 0x03 | NHP-AOP | Server→AC | AC operation request |
| 0x04 | NHP-ART | AC→Server | AC operation result |
| 0x05 | NHP-LST | Agent→Server | List services and applications |
| 0x06 | NHP-LRT | Server→Agent | Service list result |
| 0x07 | NHP-COK | Server→Agent | Cookie for re-knock |
| 0x08 | NHP-RKN | Agent→Server | Re-knock with cookie |
| 0x09 | NHP-RLY | Relay→Server | Relayed packet |
| 0x0A | NHP-AOL | AC→Server | AC online notification |
| 0x0B | NHP-AAK | Server→AC | Acknowledgment of AC online notification |
| 0x0C | NHP-OTP | Agent→Server | One-time passcode request |
| 0x0D | NHP-REG | Agent→Server | Agent registration |
| 0x0E | NHP-RAK | Server→Agent | Registration acknowledgment |
| 0x0F | NHP-ACC | Agent→AC | Access request |
| 0x10 | NHP-EXT | Agent→Server | Immediate disconnection request |

Values 0x11-0x16 are used by DHP message types in the reference implementation. They are not defined in this document. Values 0x17-0xFF are reserved.

## Message Definitions

Message bodies are JSON objects. Field names below are the JSON keys used by the reference implementation's message structures. A field is optional where the reference implementation marks it `omitempty`.

### NHP-KPL (Keepalive)

Keepalive messages maintain session state between components. This revision does not define a body structure.

### NHP-KNK (Knock)

Sent by NHP-Agent to NHP-Server to request access to a resource.

| JSON Key | Type | Required | Description |
|----------|------|----------|-------------|
| headerType | integer | Yes | Header type value |
| usrId | string | Yes | User identifier |
| devId | string | Yes | Device identifier |
| orgId | string | No | Organization identifier |
| aspId | string | Yes | Authorization Service Provider identifier |
| resId | string | Yes | Resource identifier |
| results | object | No | Results of client-side checks |
| usrData | object | No | Additional user data |

### NHP-ACK (Knock Acknowledgment)

Sent by NHP-Server to NHP-Agent in response to NHP-KNK.

| JSON Key | Type | Required | Description |
|----------|------|----------|-------------|
| errCode | string | Yes | Result code |
| errMsg | string | No | Error description |
| resHost | object | Yes | Map of resource host addresses |
| opnTime | integer | Yes | Open time for access |
| aspToken | string | No | Token for AC-side validation |
| agentAddr | string | Yes | Source address observed for the agent |
| acTokens | object | Yes | Map of AC access tokens |
| preActions | object | No | Pre-access actions |
| redirectUrl | string | No | Redirect URL |

### NHP-AOP (AC Operation Request)

Sent by NHP-Server to NHP-AC to open access for an agent.

| JSON Key | Type | Required | Description |
|----------|------|----------|-------------|
| usrId | string | Yes | User identifier |
| devId | string | Yes | Device identifier |
| orgId | string | No | Organization identifier |
| aspId | string | Yes | Authorization Service Provider identifier |
| resId | string | Yes | Resource identifier |
| srcAddrs | array of NetAddress | Yes | Source addresses to allow |
| dstAddrs | array of NetAddress | Yes | Destination addresses to allow |
| opnTime | integer | Yes | Open time |

### NHP-ART (AC Operation Result)

Sent by NHP-AC to NHP-Server with the result of NHP-AOP.

| JSON Key | Type | Required | Description |
|----------|------|----------|-------------|
| errCode | string | Yes | Result code |
| errMsg | string | No | Error description |
| opnTime | integer | Yes | Open time granted |
| token | string | Yes | AC access token |
| preAct | PreAccessInfo | No | Pre-access information |

### NHP-LST (List Request) and NHP-LRT (List Result)

NHP-LST carries the same identity fields as NHP-KNK, without a resource identifier: usrId, devId, orgId (optional), aspId, and usrData (optional).

NHP-LRT carries:

| JSON Key | Type | Required | Description |
|----------|------|----------|-------------|
| errCode | string | Yes | Result code |
| errMsg | string | No | Error description |
| list | object | No | Services and applications |

### NHP-COK (Cookie)

Sent by NHP-Server to NHP-Agent.

| JSON Key | Type | Required | Description |
|----------|------|----------|-------------|
| trxId | integer (64-bit) | Yes | Transaction identifier |
| cookie | string | Yes | Cookie for re-knock |

### NHP-RKN (Re-Knock)

Sent by NHP-Agent to NHP-Server with the cookie from NHP-COK. This revision does not define the body structure.

### NHP-RLY (Relayed Packet)

Sent by a relay to NHP-Server to forward a packet on behalf of a client.

| JSON Key | Type | Required | Description |
|----------|------|----------|-------------|
| srcAddr | NetAddress | Yes | Original client address |
| innerPkt | string | Yes | Base64-encoded inner NHP packet |

### NHP-AOL (AC Online)

Sent by NHP-AC to NHP-Server to report the resources it serves.

| JSON Key | Type | Required | Description |
|----------|------|----------|-------------|
| aspId | string | Yes | Authorization Service Provider identifier |
| resIds | array of string | Yes | Resource identifiers |
| acId | string | No | AC identifier |

### NHP-AAK (AC Acknowledgment)

Sent by NHP-Server to NHP-AC after receiving NHP-AOL.

| JSON Key | Type | Required | Description |
|----------|------|----------|-------------|
| errCode | string | Yes | Result code |
| errMsg | string | No | Error description |
| acAddr | string | Yes | AC address |

### NHP-OTP (One-Time Passcode Request)

Sent by NHP-Agent to NHP-Server.

| JSON Key | Type | Required | Description |
|----------|------|----------|-------------|
| usrId | string | Yes | User identifier |
| devId | string | Yes | Device identifier |
| orgId | string | No | Organization identifier |
| aspId | string | Yes | Authorization Service Provider identifier |
| pass | string | No | Passcode |
| pubKey | string | No | Agent public key |
| usrData | object | No | Additional user data |

### NHP-REG (Register)

Sent by NHP-Agent to NHP-Server to register its public key.

| JSON Key | Type | Required | Description |
|----------|------|----------|-------------|
| usrId | string | Yes | User identifier |
| devId | string | Yes | Device identifier |
| orgId | string | No | Organization identifier |
| aspId | string | Yes | Authorization Service Provider identifier |
| otp | string | No | One-time passcode |
| pubKey | string | No | Agent public key |
| usrData | object | No | Additional user data |

### NHP-RAK (Register Acknowledgment)

Sent by NHP-Server to NHP-Agent.

| JSON Key | Type | Required | Description |
|----------|------|----------|-------------|
| errCode | string | Yes | Result code |
| errMsg | string | No | Error description |
| aspId | string | Yes | Authorization Service Provider identifier |
| expiresAt | integer | No | Unix time, in seconds, when the registered key expires |

### NHP-ACC (Access)

Sent by NHP-Agent to NHP-AC to access a resource. The NHP-AC replies with an access acknowledgment carrying errCode, errMsg (optional), and agentAddr (optional).

| JSON Key | Type | Required | Description |
|----------|------|----------|-------------|
| usrId | string | Yes | User identifier |
| devId | string | Yes | Device identifier |
| orgId | string | No | Organization identifier |
| acToken | string | Yes | Access token from NHP-ACK |
| usrData | object | No | Additional user data |

### NHP-EXT (Disconnect)

Sent by NHP-Agent to NHP-Server to request immediate disconnection. This revision does not define the body structure.

### NetAddress

| JSON Key | Type | Required | Description |
|----------|------|----------|-------------|
| ip | string | Yes | IP address |
| port | integer | No | Port number |
| proto | string | No | "tcp" or "udp"; empty for any |

### PreAccessInfo

| JSON Key | Type | Required | Description |
|----------|------|----------|-------------|
| acIp | string | Yes | AC IP address |
| acPort | string | Yes | AC port |
| acPubKey | string | Yes | AC public key |
| acToken | string | Yes | AC access token |
| acCipherScheme | integer | Yes | Cipher scheme of the AC |


# Logging and Auditing

NHP provides comprehensive logging capabilities to support security monitoring, compliance, and forensic analysis.

## Log Types

NHP defines the following log categories:

Access Logs
: Record all access attempts, including source identity, timestamp, requested resource, and decision outcome.

Authentication Logs
: Record authentication events including key exchanges, identity verification, and authentication failures.

Policy Logs
: Record policy evaluation decisions and the factors considered.

System Logs
: Record component health, configuration changes, and operational events.

## Log Format

All NHP logs SHOULD use structured JSON format with the following mandatory fields:

~~~json
{
  "timestamp": "2025-01-01T12:00:00.000Z",
  "log_type": "access",
  "component": "nhp-ac-01",
  "session_id": "abc123...",
  "user_id": "user@example.com",
  "device_id": "device-uuid",
  "source_ip": "192.0.2.1",
  "resource_id": "resource-001",
  "action": "access_granted",
  "details": {}
}
~~~

## Log Transmission {#logging-transmission}

NHP-LOG and NHP-LAK are not implemented in the reference implementation, and this revision does not register message types for them. The requirements below are intended for a future revision that defines the log transport:

NHP-AC components transmit logs to NHP-Server. Implementations MUST:

* Encrypt all log transmissions using the established Noise session
* Batch logs to reduce network overhead
* Implement retry logic for failed transmissions
* Store logs locally if transmission fails

## Compliance Considerations

NHP logging supports compliance with:

* SOC 2 Type II audit requirements
* GDPR access logging requirements
* HIPAA audit trail requirements
* PCI-DSS logging requirements

# Integration with SDP

NHP is designed to integrate seamlessly with existing Software-Defined Perimeter (SDP) deployments as defined in {{CSA.SDP.Spec2.0}}.

## Integration Architecture

In an SDP integration, NHP components map to SDP components as follows:

| NHP Component | SDP Component |
|---------------|---------------|
| NHP-Agent | SDP Initiating Host |
| NHP-Server | SDP Controller |
| NHP-AC | SDP Gateway |

## Integration Process

1. **Discovery:** SDP Controller advertises NHP-Server endpoint to SDP Initiating Hosts.

2. **Authentication:** SDP Initiating Host uses NHP-KNK to authenticate with NHP-Server instead of SPA.

3. **Authorization:** NHP-Server queries SDP Controller for policy decisions.

4. **Enforcement:** NHP-AC opens ports on SDP Gateway based on NHP-AOP commands.

## Benefits of NHP-SDP Integration

* **Stronger Cryptography:** NHP's Noise-based key exchange provides better forward secrecy than traditional SPA.
* **Mutual Authentication:** Both client and server authenticate each other.
* **Scalability:** NHP's architecture supports enterprise-scale deployments.
* **Extensibility:** NHP message types support richer interaction patterns.

# Integration with DNS

NHP can integrate with DNS infrastructure to provide stealth resolution of protected resources.

## DNS Integration Architecture

~~~
+-------------+     +-------------+     +-------------+
| NHP-Agent   |---->| NHP-Server  |---->| DNS Server  |
|             |     |             |     | (Internal)  |
+-------------+     +-------------+     +-------------+
      |                   |
      v                   v
+-------------+     +-------------+
| Public DNS  |     | NHP-AC      |
| (No Records)|     |             |
+-------------+     +-------------+
~~~

## Integration Process

1. Protected resources have no public DNS records.
2. NHP-Agent authenticates with NHP-Server via NHP-KNK.
3. NHP-Server returns resource IP addresses in NHP-ACK only after successful authentication.
4. NHP-Agent can then connect to the resolved addresses.

This prevents DNS enumeration attacks and keeps resource addresses invisible to unauthorized users.

# Integration with FIDO

NHP supports integration with FIDO2/WebAuthn for strong user authentication.

## FIDO Integration Flow

1. User initiates NHP-KNK with FIDO assertion
2. NHP-Server validates FIDO assertion with FIDO server
3. Upon successful FIDO authentication, NHP-Server proceeds with access grant

## Recovery and Fallback

For FIDO authentication failures, NHP supports fallback to:

* One-Time Password (OTP) via NHP-OTP message
* SMS/Email verification codes
* Recovery codes

# Security Considerations

## Infrastructure Invisibility

NHP ensures infrastructure invisibility by:

* Encrypting all control plane traffic using Noise Protocol
* Requiring mutual authentication before any resource visibility
* Maintaining default-deny firewall rules on all NHP-AC components
* Supporting ephemeral port allocation for data plane connections

## Replay Attack Prevention

NHP prevents replay attacks through:

* Timestamp validation with configurable tolerance (RECOMMENDED: 60 seconds)
* Unique nonce per message
* Session-bound tokens that cannot be reused across sessions

## Key Security

Implementations MUST:

* Use cryptographically secure random number generators for all key generation
* Store private keys in secure enclaves or HSMs where available
* Implement key rotation policies
* Securely erase key material when no longer needed

## Session Security

* Sessions MUST have configurable expiration (RECOMMENDED default: 4 hours)
* Sessions MUST be revocable by NHP-Server
* Session tokens MUST be bound to client identity and IP address

## Denial of Service Mitigation

NHP provides DoS resistance through:

* Cryptographic puzzles for computationally expensive operations
* Rate limiting on NHP-Server and NHP-AC
* Cookie-based session resumption to avoid repeated handshakes

## Limitations

NHP does not protect against:

* Compromised endpoints with valid credentials
* Insider threats with legitimate access
* Attacks on the data plane after access is granted
* Social engineering attacks targeting user credentials

# IANA Considerations

This document requests IANA to establish a new registry named "NHP Message Types" with an 8-bit type field. The registration policy is Specification Required {{RFC8126}}. The initial values are:

| Value | Name | Reference |
|-------|------|-----------|
| 0x00 | NHP-KPL | This document |
| 0x01 | NHP-KNK | This document |
| 0x02 | NHP-ACK | This document |
| 0x03 | NHP-AOP | This document |
| 0x04 | NHP-ART | This document |
| 0x05 | NHP-LST | This document |
| 0x06 | NHP-LRT | This document |
| 0x07 | NHP-COK | This document |
| 0x08 | NHP-RKN | This document |
| 0x09 | NHP-RLY | This document |
| 0x0A | NHP-AOL | This document |
| 0x0B | NHP-AAK | This document |
| 0x0C | NHP-OTP | This document |
| 0x0D | NHP-REG | This document |
| 0x0E | NHP-RAK | This document |
| 0x0F | NHP-ACC | This document |
| 0x10 | NHP-EXT | This document |

Values 0x11-0x16 are used by DHP message types in the reference implementation and are to be registered by the document that defines them. Values 0x17-0xFF are reserved for future use.

# Reference Implementation

An open-source reference implementation of NHP is available at:

https://github.com/OpenNHP/opennhp

A live demo of the protocol is available at:

https://opennhp.org/demo/

## Implementation Characteristics

The OpenNHP reference implementation is designed with the following characteristics:

### Memory-Safe Language

OpenNHP is implemented in **Go (Golang)**, a memory-safe programming language that eliminates entire classes of vulnerabilities common in C/C++ implementations:

* **No Buffer Overflows:** Go's built-in bounds checking prevents buffer overflow attacks.
* **No Use-After-Free:** Automatic garbage collection eliminates dangling pointer vulnerabilities.
* **No Null Pointer Dereferences:** Go's type system and nil handling prevent null pointer crashes.
* **Race Condition Detection:** Built-in race detector helps identify concurrency issues during development.

This choice aligns with recommendations from CISA, NSA, and other security agencies advocating for memory-safe languages in critical infrastructure software.

### Cross-Platform Support

OpenNHP provides native support across multiple platforms:

| Platform | Components | Description |
|----------|------------|-------------|
| Linux | Agent, Server, AC | Full production support for x86_64, ARM64 |
| Windows | Agent, Server, AC | Native Windows service integration |
| macOS | Agent | Desktop client with system integration |
| FreeBSD | Agent, Server, AC | BSD-family operating system support |
| Android | Agent (Library) | Mobile SDK for Android applications |
| iOS | Agent (Library) | Mobile SDK for iOS applications |

### Modular Architecture

The implementation provides separate binaries for each NHP component:

* **nhp-agent:** Client-side agent for initiating NHP connections
* **nhp-server:** Control plane server for authentication and authorization
* **nhp-ac:** Access controller for policy enforcement

Each component can be deployed independently, enabling flexible deployment topologies from standalone to distributed enterprise configurations.

### Cryptographic Implementation

The reference implementation uses well-audited cryptographic libraries:

* **Noise Protocol:** flynn/noise library for Noise Framework handshakes
* **Curve25519:** golang.org/x/crypto for elliptic curve operations
* **ChaCha20-Poly1305:** Standard library crypto/cipher for AEAD encryption
* **HKDF:** golang.org/x/crypto/hkdf for key derivation

### Performance Characteristics

The Go implementation provides:

* **Low Latency:** Typical NHP handshake completes in under 10ms on local networks
* **High Throughput:** Single NHP-Server can handle thousands of concurrent sessions
* **Minimal Footprint:** Agent binary under 15MB, low memory consumption
* **Concurrent Design:** Goroutine-based concurrency for efficient resource utilization

### Open Source Governance

The OpenNHP project operates under the Apache 2.0 license, fostering community collaboration and transparent development to accelerate adoption and ensure rigorous peer review of its security mechanisms.

## Practical Use Case: StealthDNS

StealthDNS is a Zero Trust DNS client powered by OpenNHP that demonstrates practical application of the NHP protocol for DNS-level infrastructure hiding. It is available at:

https://github.com/OpenNHP/StealthDNS

StealthDNS implements the NHP-DNS integration described in this specification, providing:

* **Invisible DNS Resolution:** Protected domains have no public DNS records. Only authenticated clients can resolve hidden service addresses.

* **NHP-Powered Authentication:** Uses the OpenNHP library to perform cryptographic NHP knocking before DNS resolution.

* **Transparent Local Resolver:** Runs as a local DNS resolver (127.0.0.1:53), requiring no application changes.

* **Cross-Platform Support:** Available on Windows, macOS, Linux, Android, and iOS.

The StealthDNS workflow demonstrates the authenticate-before-connect principle:

1. Application performs DNS lookup for a protected domain.
2. StealthDNS checks if the domain is NHP-protected.
3. If protected, StealthDNS performs NHP knock with identity and device context.
4. Upon successful authentication, the NHP Controller returns ephemeral address mappings.
5. StealthDNS returns valid DNS records only to authorized clients.
6. Unauthorized clients receive NXDOMAIN—the service remains invisible.

This enforces **identity before visibility** and **authorization before connectivity**, demonstrating real-world application of NHP principles.

--- back

# Acknowledgments
{:numbered="false"}

This work builds upon foundational research from the Cloud Security Alliance (CSA) Zero Trust Working Group, particularly the "Stealth Mode SDP for Zero Trust Network Infrastructure" whitepaper {{CSA.NHP.Whitepaper}}. The authors acknowledge the contributions of the CSA Zero Trust Research Working Group.

The authors would also like to thank the China Computer Federation (CCF) for their collaborative support, and the OpenNHP open source community for their contributions, testing, and feedback on early implementations of the Network-Infrastructure Hiding Protocol.



