# IETF Internet-Draft: Network infrastructure Hiding Protocol (NHP)

This repository hosts the working draft and associated materials for the proposed IETF Internet-Draft:

> **Network infrastructure Hiding Protocol (NHP)**  
> Draft name: `draft-ietf-secdispatch-nhp.md`  
> Category: `Informational`  
> Submission Type: `IETF`  
> Area: `SEC`  
> Working Group: `SECDispatch`  

## Overview

The **Network infrastructure Hiding Protocol (NHP)** is a cryptography-driven, session-layer protocol designed to implement Zero Trust principles by **hiding network infrastructure** from unauthorized actors. NHP prevents exposure of IP addresses, ports, and domains until authentication is complete — enforcing an "authenticate-before-connect" model at OSI layers 5–6.

NHP is especially relevant in the era of **AI-driven cyber threats**, where automated tools can scan, identify, and exploit vulnerabilities at machine speed. In this “Dark Forest” landscape, **visibility equates to vulnerability** — NHP turns that visibility off by default.

## Key Concepts

- **Session-layer protocol**: operates at OSI layers 5–6.
- **Mutual authentication** before TCP/TLS handshakes.
- **Noise Protocol Framework** and **Curve25519 ECC**.
- **Defense by invisibility**: unauthorized users see nothing.
- **Designed for Zero Trust** and integrates with SDP, FIDO, and DNS.

## Repository Contents

- `draft-nhp-protocol-00.md` – Markdown source of the IETF draft.
- `README.md` – This file.
- `references/` – Supporting links and citations.
- `diagrams/` – Protocol architecture and workflow illustrations.
- `LICENSE` – Apache 2.0 license.

## Draft Status

The current draft version is `00` (as of July 22, 2025). The document is being prepared for submission via the [IETF Independent Submission stream](https://www.ietf.org/about/groups/independent/).  

You can preview the current version at:  
👉 [Latest HTML Draft](https://example.com/LATEST)


## Contributing

We welcome contributions to improve the draft:

- Submit issues for typos, clarity, or technical improvements.
- Open pull requests for draft changes (Markdown preferred).
- Join the discussion via email: [opennhp@gmail.com](mailto:opennhp@gmail.com)

## Acknowledgments

This draft builds upon prior work from the [Cloud Security Alliance](https://cloudsecurityalliance.org/), the [China Computer Federation](https://www.ccf.org.cn/en/), and the [OpenNHP](https://github.com/OpenNHP/opennhp) open source project. Special thanks to contributors and reviewers across the Zero Trust and network security communities.

---

© 2025 OpenNHP. Licensed under the Apache 2.0 License.

