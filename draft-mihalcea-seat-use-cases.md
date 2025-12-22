---
title: "Use Cases and Properties for Integrating Remote Attestation with Secure Channel Protocols"
abbrev: "SEAT Use Cases"
category: info

docname: draft-mihalcea-seat-use-cases-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: Security
workgroup: SEAT Working Group
keyword:
  - remote attestation
  - TLS
  - confidential computing
  - IoT
  - RATS
venue:
group: SEAT
type: Working Group
mail: seat@ietf.org

author:
  - fullname: Ionuț Mihalcea
    organization: Arm
    email: ionut.mihalcea@arm.com
  - fullname: Muhammad Usama Sardar
    organization: TU Dresden
    email: muhammad_usama.sardar@tu-dresden.de
  - fullname: Thomas Fossati
    organization: Linaro
    email: thomas.fossati@linaro.org
  - fullname: Yuning Jiang
    email: jiangyuning2@h-partners.com
  - fullname: Meiling Chen
    organization: China Mobile
    email: chenmeiling@chinamobile.com

normative:

informative:
    RFC9334: rats-arch
    I-D.draft-ccc-wimse-twi-extensions: wimse-twi
    I-D.draft-ietf-rats-eat-measured-component: rats-measured
    Meeting-122-TLS-Slides:
     title: "Identity Crisis in Attested TLS for Confidential Computing"
     date: 20 March 2025,
     target: https://datatracker.ietf.org/meeting/122/materials/slides-122-tls-identity-crisis-00
     author:
     - ins: M. U. Sardar
     - ins: M. Moustafa
     - ins: T. Aura
    AI-agents:
     title: "AI agents that matter"
     date: 1 July 2024,
     target: https://arxiv.org/abs/2407.01502
     author:
     - ins: S. Kapoor
     - ins: B. Stroebl
     - ins: Z. S. Siegel
     - ins: N. Nadgir
     - ins: A. Narayanan

--- abstract

This document outlines use cases and desirable properties for integrating remote
attestation (RA) capabilities with secure channel establishment protocols, with
an initial focus on Transport Layer Security (TLS) v1.3 Handshake. Traditional
peer authentication in TLS establishes trust in a peer's network identifiers but
provides no assurance regarding the integrity of its underlying software and
hardware stack. Remote attestation addresses this gap by enabling a peer to
provide verifiable evidence about its current state, including the state of its
trusted computing base (TCB). This document explores specific use cases, such as
confidential data collaboration and secure secrets provisioning, to motivate the
need for this integration. From these use cases, it specifies a set of essential
properties the protocol solution must have, including cryptographic binding to
the TLS connection, evidence freshness, and flexibility to support different
attestation models. This document is intended to serve as an input to the design
of protocol solutions within the SEAT working group.

--- middle

# Introduction

## Establishing Trust in Secure Communications

Traditional secure channel protocols, such as Transport Layer Security (TLS),
primarily establish trust in a peer's identity. This is typically achieved
through mechanisms like a Public Key Infrastructure (PKI), where a trusted
Certification Authority (CA) vouches for the binding between a public key and an
identifier (e.g., a hostname).

However, this model has a core limitation: identity authentication provides no
assurance about the peer's internal state or the integrity of its software
stack. A compromised server, for instance, can still present a valid X.509
certificate and be considered "trusted" by a client. This gap allows compromised
endpoints to maintain network access and the trust of their peers, posing a
significant security risk in many environments.

## The Role of Remote Attestation

Remote Attestation (RA), as described in the RATS architecture {{RFC9334}}, is a
mechanism designed to fill this gap. RA allows an entity (the "Attester") to
produce verifiable "Evidence" about its current runtime state. This Evidence
covers the Attester's TCB, and can thus include measurements of its firmware,
operating system, and application code, as well as the configuration of its
hardware and software security features (e.g., secure boot status, memory
isolation). A "Relying Party" can then use this Evidence, often with the help of
a trusted "Verifier", to appraise the Attester's trustworthiness.

By integrating RA into a secure channel establishment protocol, a second
dimension of trust—trustworthiness—is added to complement regular peer
authentication. This allows a peer to make authorization decisions based not
just on who the other party is, but also on what it is (e.g., an AMD
SEV-SNP-based server running in some known datacenter) and whether its state is
acceptable.

## Purpose and Scope

The purpose of this document is to outline the key use cases that motivate the
integration of RA with secure channel protocols and to establish a set of
essential properties for such an integration. The initial focus is on TLS 1.3
and its datagram-oriented variant, DTLS 1.3.

This document is intended as an input to the design of protocol solutions within
the SEAT working group. It defines the "why" and the "what" (the requirements),
but not the "how" (the protocol specification itself). A key goal is to define
requirements for a solution that is agnostic to any specific attestation
technology (e.g., Trusted Platform Modules (TPMs), Intel TDX, AMD SEV, Arm CCA).

# Terminology

This document uses the terminology defined in the RATS Architecture {{RFC9334}},
including "Attester", "Relying Party", "Verifier", "Evidence", and "Attestation
Results".

This document also uses the following terms:

* Trusted Computing Base (TCB) of a device: all security-relevant components:
  hardware, firmware, software, and their respective configurations.
* Confidential Workload: as defined in {{-wimse-twi}}.
* Measurements: as defined in {{-rats-measured}}.
* AI agent: An AI agent is a software principal (typically long-running) that performs
closed-loop "perceive -> plan -> act" cycles using an LLM or other model,
and invokes external tools/APIs that may read sensitive data or change
system/network state. Its configuration (e.g., model choice, tool enablement,
prompt template) can change independently of the binary/image and usually
more frequently than typical platform TCB updates {{AI-agents}}.

# Use Cases

This section provides the concrete motivation for the WG's work by describing
specific use cases. For each case, the scenario, actors, and specific security
guarantees needed from RA are described.

## Secure Provisioning and High-Assurance Operations

Goal: Ensure the integrity of workloads and devices when bootstrapping their
identity or receiving critical commands.

Use case: Runtime Secret Provisioning: A confidential workload starts in a
generic state and needs to fetch secrets (e.g., API keys, database credentials,
encryption keys) to become operational.

* Requirement: The workload must attest its runtime state (TEE genuineness,
  software measurements) to a secrets management service. The service will only
  release the secrets after successful verification, ensuring they are delivered
  exclusively to a trustworthy environment. This use-case also covers secure
  device onboarding for IoT devices that lack a pre-provisioned identity.

Use case: High-Assurance Command Execution: An operator sends a critical command
to a remote system (e.g., an industrial controller, a financial transaction
processor).

* Requirement: The system must provide fresh attestation Evidence to the
  operator to prove its integrity before the command is dispatched. This
  prevents commands from being executed on a compromised system.

## Confidential Data Collaboration

Goal: Enable multiple parties to collaborate on sensitive, combined datasets
without exposing raw data to each other or to the infrastructure operator.

Use case: Data Clean Rooms: Multiple data providers contribute sensitive data to
a confidential workload for joint analysis. Data consumers receive aggregated
insights without ever accessing the raw, combined dataset.

* Requirement: Before sending data, each data provider must attest the
  confidential workload to verify it is running the authorized analysis code in
  a secure Trusted Execution Environment (TEE). Similarly, data consumers must
  attest the workload to trust the integrity of the results.

Use case: Secure Multi-Party Computation (MPC): Distributed parties
collaboratively compute a function (e.g., train a machine learning model)
without sharing their local data.

* Requirement: The central aggregator, as well as each participating client,
  must be able to mutually attest to ensure all parties are running the correct,
  untampered MPC algorithm in a trusted environment.

## Network Infrastructure Integrity

Goal: Verify the integrity of network devices that form the foundation of
communication.

Use case: Attestation of Network Functions: A router, switch, or firewall joins
a network's management plane. A Virtualized Network Function (VNF) is
instantiated on a generic server.

* Requirement: The network orchestrator must verify the device's integrity
  (e.g., secure boot enabled, running signed OS and firmware) before allowing it
  to join the network and receive policy. This prevents a compromised router
  from misdirecting traffic or a malicious VNF from inspecting sensitive
  packets.

Use case: Securing Control and Management Planes: An administrator connects to a
network device's management interface.

* Requirement: The administrator's client must verify the integrity of the
  management endpoint on the network device to ensure they are not connecting to
  a compromised interface that could steal credentials or manipulate the device.

## Operation-Triggered Attestation for High-Impact Application Operations

Goal: Ensure the integrity of application services at operation time,
when security posture may change after initial channel establishment.

Use case: High-Assurance Operation Execution in Dynamic Application Services:
An application service instance (e.g., AI agent) or confidential computing
environment (which could host an AI agent) maintains a (D)TLS connection with
a peer and must execute a high-impact action (e.g., payment initiation,
configuration change, privileged command).

* Requirement 1: Before executing a high-impact operation over the existing
connection, the peer must present fresh, connection-bound attestation evidence
reflecting the current behavior-affecting posture (e.g., enabled capabilities,
policy configuration, runtime permissions).

* Requirement 2: The mechanism should support lightweight, dynamic attestation
within the existing connection, without necessarily requiring a full new TLS
handshake, so that behavior-affecting posture changes are visible to relying
parties when required by local policy.

## Trusted Software and Firmware Upgrade

Goal: Ensure integrity during software/firmware updates, guaranteeing the
reliability and security of network services.

Contrast with infrastructure integrity use case: Trusted upgrade ensures 
"the right thing is installed during the upgrade process," while network
infrastructure integrity ensures "the entire network operates securely
as expected during the initialization phase."

Use case: Integrity Attestation During Software/Firmware Upgrade Deployment:
A network orchestrator pushes new software or firmware to network devices
(e.g., routers, switches, or virtual network function instances).
The management system must continuously verify the integrity state of the
target device before, during, and after the upgrade, including
rollback procedures.

From attested TLS perspective, network device (acting as TLS client)
is Attester and network orchestrator (acting as TLS server) is Verifying
RP.

* Requirement 1: Before initiating the upgrade, the target device must
be verified to be in a known, healthy baseline state (e.g., secure
boot enabled, running a signed software/firmware version).

* Requirement 2: The software/firmware package must be fetched from a
trusted source via a secure channel with attestation capabilities
(e.g., TLS with remote attestation extensions).

* Requirement 3: Upon completion, the integrity measurements of the
new instance (e.g., software hash, firmware manifest) must be verified.
The device can only be admitted to the service plane after compliance
with the security policy is confirmed.

* Requirement 4: If post-upgrade verification fails, an automated and
verified rollback to a previously known good version must be triggered
based on attestation evidence.

# Integration Properties

This section provides a list of desirable properties for designs that integrate
RA into secure channel protocols. Proposed integration protocols should make it
clear which of these properties are fulfilled, and how.

## Cryptographic Binding to Communication Channel

The attestation Evidence or Attestation Result is cryptographically bound to the
specific secure channel instance (e.g., the TLS connection). This prevents replay
and relay attacks where an attacker presents valid, but old or unrelated
Evidence from a different connection or context. This binding is paramount for all
use cases.

## Cryptographic Binding to Machine Identifier

Evidence should be cryptographically bound to the identifier provided to the machine by the infrastructure provider to prevent diversion attacks {{Meeting-122-TLS-Slides}}.

## Attestation Credential Freshness

The Relying Party is able to verify that the Evidence or Attestation Result it
receives was freshly generated by the Attester for the current connection. State is
transient, and credentials from a previous connection may no longer be valid. See
{{Section 10 of -rats-arch}} for more details about freshness in the context of
RA.

## Negotiation and Capability Discovery

Peers have a secure mechanism to discover each other's support for RA, the
specific attestation formats they can produce or consume, and the attestation
models they support. This enables interoperability and allows for graceful
fallback for endpoints that do not support RA.

## Attestation Model Flexibility

The solution supports both the Background Check and Passport models as defined
in the RATS architecture {{RFC9334}}. The Background Check model is essential
for use cases requiring maximum freshness, while the Passport model is better
suited for performance, scalability, and scenarios where the Verifier may be
offline or unreachable by the Relying Party.

## Interaction with Peer Authentication

The solution supports using RA in conjunction with traditional PKI-based
authentication (e.g., X.509 certificates). This provides two independent pillars
of trust: trustworthiness (from RA) and identity (from PKI). The solution may
also support RA as the sole method of authentication in constrained use cases,
such as device onboarding, where a device has no stable, long-term identity yet.
This latter option could have a negative impact on the security of the overall
design, warranting additional security considerations.

## Credential Lifecycle Management

The solution can provide a mechanism for re-attesting or refreshing attestation
credentials on an existing, long-lived connection without requiring a full new
handshake. For long-lived connections, the initial attestation may become stale,
and a lightweight refresh mechanism is beneficial towards re-evaluating the
peer's state.

## Privacy Preservation

The solution must not degrade the privacy of a standard TLS connection. Evidence
can contain highly specific, unique information about a device's hardware and
software, which could be used as an advanced tracking mechanism, following a
user across different connections and services. The design must consider how to
minimize this leakage, especially when a third-party Verifier is involved in the
protocol exchange.

## Performance and Efficiency

The introduction of attestation should not add prohibitive latency or overhead
to the connection establishment process. To be widely adopted, the solution must
be practical. While some overhead is unavoidable, multiple additional
round-trips or very large payloads in the initial handshake should be minimized.

# Security Considerations

This document describes use cases and integration properties. The security of
any protocol designed to fulfill these properties will depend on its specific
mechanisms. However, any solution must address the following high-level
considerations:

* Replay and Relay Protection: The requirements for cryptographic binding and
  freshness are critical. Failure to bind attestation credentials tightly to the
  current connection would allow an adversary to replay or relay old or stolen, yet
  valid credentials from a compromised system, completely undermining the
  security goals.

* Verifier Trust and Privacy: In the Background Check model, the Relying Party
  communicates with a Verifier. This reveals to the Verifier that the Relying
  Party is communicating with the Attester. Depending on the scenario, this
  could leak sensitive information about business relationships or user
  activity. Solutions should consider mechanisms to minimize the data revealed
  to the Verifier.

* Downgrade Attacks: The negotiation of attestation capabilities must be secure.
  An active attacker must not be able to trick two parties that both support
  attestation into negotiating a connection without it.

* Evidence Semantics: This document does not define attestation appraisal
  policies. However, a Relying Party must be careful when interpreting
  Attestation Results. A "valid" attestation only means the Evidence is
  authentic and correctly signed; it does not automatically mean the underlying
  system is "secure". The Relying Party must have a clear policy for what
  measurements, software versions, and security configurations are acceptable.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

TODO
