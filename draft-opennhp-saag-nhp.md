---
title: "Network-Infrastructure Hiding Protocol"
abbrev: "NHP"
category: info
docname: draft-opennhp-saag-nhp-latest
submissiontype: independent
number: 02
date: 2025-01-01
v: 3
area: "Security"
workgroup: "saag"
keyword:
 - zero trust
 - session layer
 - network obfuscation
 - SDP
venue:
  group: "SAAG"
  type: "Working Group"
  mail: "saag@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/saag/"
  github: "OpenNHP/ietf-rfc-nhp"
  latest: "https://OpenNHP.github.io/ietf-rfc-nhp/draft-opennhp-saag-nhp.html"

author:
 -
    fullname: Benfeng Chen
    organization: OpenNHP
    email: benfeng@gmail.com

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

9. **Logging and Audit:** NHP-AC uploads access logs via NHP-LOG messages for compliance and auditing.

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
    |                    |<-- NHP-LOG --------|                   |
    |                    |--- NHP-LAK ------->|                   |
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

The NHP message header is 32 bytes with the following structure:

~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|    Version    |     Type      |     Flags     |   Reserved    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                          Nonce (96 bits)                      +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                       Timestamp (64 bits)                     +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|        Payload Length         |        Header Checksum        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~

### Header Fields

Version (8 bits)
: Protocol version. Current version is 0x01.

Type (8 bits)
: Message type code. See {{message-types}}.

Flags (8 bits)
: Control flags:
  * Bit 0: Compression enabled
  * Bit 1: Fragmentation flag
  * Bit 2: Priority message
  * Bits 3-7: Reserved

Reserved (8 bits)
: Reserved for future use. MUST be set to zero.

Nonce (96 bits)
: Unique nonce for AEAD encryption. MUST be unique per message within a session.

Timestamp (64 bits)
: UNIX epoch time in milliseconds. Used for replay protection.

Payload Length (16 bits)
: Length of the encrypted payload in bytes.

Header Checksum (16 bits)
: CRC-16 checksum of the header for integrity verification.

## Message Types {#message-types}

| Type Code | Name | Direction | Description |
|-----------|------|-----------|-------------|
| 0x00 | NHP-KPL | Any | Keepalive message |
| 0x01 | NHP-KNK | Agent→Server | Knock request |
| 0x02 | NHP-ACK | Server→Agent | Knock acknowledgment |
| 0x03 | NHP-AOP | Server→AC | AC operation command |
| 0x04 | NHP-ART | AC→Server | AC operation result |
| 0x05 | NHP-LST | Agent→Server | Resource list request |
| 0x06 | NHP-LRT | Server→Agent | Resource list response |
| 0x07 | NHP-COK | Server→Agent | Cookie for session resumption |
| 0x08 | NHP-RKN | Agent→Server | Re-knock with cookie |
| 0x09 | NHP-RLY | Relay→Server | Relayed message |
| 0x0A | NHP-AOL | AC→Server | AC online notification |
| 0x0B | NHP-AAK | Server→AC | AC acknowledge |
| 0x0C | NHP-OTP | Any | One-time password request |
| 0x0D | NHP-REG | Agent→Server | Public key registration |
| 0x0E | NHP-RAK | Server→Agent | Registration acknowledgment |
| 0x0F | NHP-ACC | Agent→AC | Access request |
| 0x10 | NHP-LOG | AC→Server | Log upload |
| 0x11 | NHP-LAK | Server→AC | Log acknowledgment |

## Message Definitions

### NHP-KPL (Keepalive)

Keepalive messages maintain session state between components. The payload contains:

| Field | Size | Description |
|-------|------|-------------|
| Session ID | 16 bytes | Current session identifier |
| Sequence | 4 bytes | Monotonic sequence number |

### NHP-KNK (Knock)

The knock message initiates access request from NHP-Agent to NHP-Server. The encrypted payload contains:

| Field | Size | Description |
|-------|------|-------------|
| User ID | Variable | Unique user identifier |
| Device ID | Variable | Unique device identifier |
| Device Fingerprint | 32 bytes | Device attestation hash |
| Requested Resources | Variable | List of resource identifiers |
| Context Data | Variable | Additional context (location, etc.) |

### NHP-ACK (Acknowledge)

The acknowledge message confirms knock success and provides access parameters:

| Field | Size | Description |
|-------|------|-------------|
| Status Code | 2 bytes | Result status |
| Session ID | 16 bytes | Assigned session identifier |
| Access Token | Variable | Token for NHP-AC access |
| AC Addresses | Variable | List of AC endpoints |
| Expiration | 8 bytes | Session expiration timestamp |
| Granted Resources | Variable | List of granted resource access |

### NHP-AOP (AC Operations)

The AC operations message instructs NHP-AC to modify access rules:

| Field | Size | Description |
|-------|------|-------------|
| Operation | 1 byte | OPEN (0x01) or CLOSE (0x02) |
| Agent Address | Variable | Source IP/port of authorized agent |
| Resource ID | Variable | Target resource identifier |
| Expiration | 8 bytes | Rule expiration timestamp |
| Access Token | Variable | Token for agent verification |

### NHP-ART (AC Result)

The AC result message reports operation status:

| Field | Size | Description |
|-------|------|-------------|
| Status Code | 2 bytes | Operation result |
| Operation ID | 16 bytes | Reference to NHP-AOP |
| Details | Variable | Additional status information |

### NHP-ACC (Access)

The access message is sent from NHP-Agent to NHP-AC to initiate data plane access:

| Field | Size | Description |
|-------|------|-------------|
| User ID | Variable | User identifier |
| Device ID | Variable | Device identifier |
| Access Token | Variable | Token from NHP-ACK |
| Requested Service | Variable | Target service identifier |

### NHP-REG (Register)

The registration message registers NHP-Agent public key with NHP-Server:

| Field | Size | Description |
|-------|------|-------------|
| User ID | Variable | User identifier |
| Device ID | Variable | Device identifier |
| Public Key | 32 bytes | Agent's static public key |
| OTP | Variable | One-time password for verification |

### NHP-RAK (Register Acknowledge)

Confirms successful registration:

| Field | Size | Description |
|-------|------|-------------|
| Status Code | 2 bytes | Registration result |
| Server Public Key | 32 bytes | Server's static public key |
| Certificate | Variable | Optional server certificate |

### NHP-LOG (Log)

Log upload message from NHP-AC to NHP-Server:

| Field | Size | Description |
|-------|------|-------------|
| AC ID | Variable | Access controller identifier |
| Log ID | 32 bytes | Unique log identifier (hash) |
| Log Content | Variable | Compressed log entries |

### NHP-LAK (Log Acknowledge)

Confirms log receipt:

| Field | Size | Description |
|-------|------|-------------|
| Log ID | 32 bytes | Received log identifier |

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

## Log Transmission

NHP-AC components transmit logs to NHP-Server using NHP-LOG messages. Implementations MUST:

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

This document requests IANA to establish a new registry for NHP Message Types with the following initial values:

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
| 0x10 | NHP-LOG | This document |
| 0x11 | NHP-LAK | This document |

Values 0x12-0xFF are reserved for future use.

# Reference Implementation

An open-source reference implementation of NHP is available at:

https://github.com/OpenNHP/opennhp

The reference implementation is developed as an open-source initiative under the OpenNHP project, fostering community collaboration and transparent development to accelerate adoption and ensure rigorous peer review of its security mechanisms.

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



