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
  -
    fullname: Tirumaleswar Reddy
    organization: Nokia
    email: "kondtir@gmail.com"
  - fullname: Yuning Jiang
    email: jiangyuning2@h-partners.com
  - fullname: Meiling Chen
    organization: China Mobile
    email: chenmeiling@chinamobile.com

normative:

informative:
    RFC9334: rats-arch
    RFC9261:
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
    MigTD:
     title: Intel TDX Migration TD
     target: https://github.com/intel/MigTD

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

# Architectural Integration Framework

To design robust protocols integrating Remote Attestation (RA) with secure
channel establishment (such as TLS 1.3 or DTLS 1.3), the required primitives
must be categorized based on their architectural interaction with the underlying
protocols. This section categorizes the required primitives based on temporal
(when attestation occurs relative to the secure channel state machine),
topological (who attests), and authentication (how integration with
authentication is handled) characteristics. Application-layer business cases are
given as examples of these underlying architectural requirements.

## Temporal Integration Patterns

Temporal integration defines the precise state machine phase where the protocol
produces, transmits, and verifies RA Evidence. It differentiates between
transport-layer gating and application-layer gating.

### In-Band Handshake Attestation

Definition: Attestation Evidence is exchanged concurrently with the secure
channel establishment. Verification of this Evidence is a strict prerequisite
for the completion of the transport key schedule and the transition to the
application data phase.

Protocol Requirements:

* The secure channel handshake must support the encapsulation of RA Evidence
  (e.g., via TLS extensions).

* The transport state machine must halt or cleanly abort (e.g., via a defined
  fatal alert) if verification fails.

* Evidence must be cryptographically bound to the handshake transcript to
  prevent relay attacks.

Instantiation Examples:

* Attested Certificate Keys: A server that claims its TLS private key is sealed
  in a secure element proves that claim inside the initial handshake before the
  client allows application data keys to be derived.

* Runtime Secret Provisioning (shared with {{immediate-post}}): A secrets
  manager relies on an authenticated and attested secure channel as a
  pre-requisite for releasing credentials or cryptographic keys.

* Confidential Data Collaboration (shared with {{immediate-post}}): Data may
  only be sent to or accepted from peers that have proven they're running
  acceptable software.

* Network Infrastructure Integrity (shared with {{immediate-post}}): A VNF must
  attest its secure boot state to an orchestrator before the orchestrator
  allocates application-layer state or transmits sensitive routing
  configurations.

### Immediate Post-Handshake Attestation {#immediate-post}

Definition: The secure channel is fully established without modification to its
core handshake. Immediately following establishment, attestation credential is
exchanged over the encrypted channel before any functional application data is
permitted to flow.

Protocol Requirements:

* The transport handshake remains standard.

* Cryptographic binding relies on deriving material from the established
  transport channel (e.g., utilizing TLS Exported Authenticators {{RFC9261}}) to
  prove the credential originates from the terminating endpoint.

* Application-layer state machines must enforce a blocking phase during which
  only attestation payloads are processed. Alternatively, the secure channel
  protocol must be wrapped in a shim layer which enforces this in-between state,
  before relinquishing control the connection to the application layer.

Instantiation Examples:

* Legacy Transport Compatibility: A deployment requires attestation but
  traverses middleboxes or uses TLS libraries that do not support custom
  handshake extensions. The application layer handles the RA exchange
  immediately after the standard TLS Finished message using Exporters to bind
  the credential to the channel.

### Deferred and Periodic Runtime Attestation

Definition: The exchange of attestation credentials occurs asynchronously after
the initial secure channel, when application sessions are established and active.

Protocol Requirements:

* Mechanisms to trigger an attestation request asynchronously over an active
  connection.

* Cryptographic binding of the runtime Evidence to the existing channel state.

* Execution must not disrupt or pause the concurrent multiplexing of
  application data streams.

Instantiation Examples:

* Operation-Triggered Attestation: An AI agent connected to a remote API must
  provide fresh credentials of its state before the API authorizes a high-privilege
  action.

* Confidential Data Collaboration: A data provider requests re-attestation from
  the confidential analytics service before uploading each new dataset or model
  update to ensure authorized code and policy remain in place.

* High-Assurance Command Execution: An operator connects to a management plane,
  then requests fresh exporter-bound credentials over the encrypted channel before
  sending critical commands.

## Topological Integration Patterns

Topological integration dictates the directionality of the trust evaluation and
defines which peer acts as the Attester and which acts as the Relying Party.

### Unidirectional Attestation

Definition: Only one peer (either the client or the server) provides
attestation credentials to the other peer.

Protocol Requirements:

* The protocol must support asymmetric payload capabilities where the Attester
  can transmit potentially large credentials, while the Relying Party requires
  only the capacity to parse and appraise (or forward to a Verifier).

* Downgrade protection to ensure an active attacker cannot suppress the Relying
  Party's demand for attestation credentials.

Instantiation Examples:

* Client Attestation: A bare-metal device connecting to a cloud provider proves
  its TCB state to the cloud provider, but the device inherently trusts the
  cloud provider's standard TLS certificate.

* Server Attestation: A client connects to an untrusted edge computing node and
  requires the edge node to prove it is running an authorized software stack.

* Secrets Provisioning: A confidential workload or IoT device attests to a
  secrets management service before the service releases credentials needed to
  bootstrap the device.

* Administrative Control Access: An operator's client requests attestation from
  a network device's management endpoint before issuing credential changes,
  while the device is satisfied with standard PKI validation of the client.

### Mutual Attestation

Definition: Both the client and the server simultaneously exchange and verify
each other's attestation credentials.

Protocol Requirements:

* Bidirectional exchange mechanisms that do not create state machine deadlocks.

* Prevention of cross-protocol attacks, ensuring that the attestation Evidence
  provided by Peer A is cryptographically bound to Peer A's specific role
  (client or server) in the session.

Instantiation Examples:

* Platform-to-Platform Migration: In confidential computing, a workload
  migrating between two hardware nodes requires mutual assurance. The
  destination attests it can securely host the workload; the source attests the
  workload's legitimate origin.

* Confidential Data Collaboration (MPC): Clients connecting to a secure
  aggregator must mutually attest to ensure all parties are running untampered
  multiparty computation algorithms.

* Data Clean Rooms: Data providers and the confidential analytics service
  mutually attest to verify both sides are running the authorized code and
  policies before any combined dataset is processed.

## Authentication Integration Patterns

Authentication integration dictates how the attestation credentials relates to
traditional authentication models (e.g., PKI).

### Composite Authentication

Definition: Attestation credential is utilized strictly to augment standard
authentication mechanisms. The channel requires proof of private key possession
and proof of the operational environment securing that key.

Protocol Requirements:

* The Evidence must explicitly cover the security state of the hardware module
  or platform holding the connection's authentication private key.

* The protocol must cryptographically bind the standard authentication checks
  (e.g., TLS CertificateVerify signature verification) with the attestation
  Evidence.

Instantiation Examples:

* Attestation of Certificate Private Key: A Relying Party requires assurance
  that the peer's TLS private key is non-exportable and resides in a
  hardware-backed secure element, augmenting the standard X.509 validation path.

* Network Infrastructure Integrity: A router or firewall proves that the TLS
  key used on its management interface is sealed to a secure element on a
  measured platform before it is allowed to join the control plane.

* High-Assurance Operations: A service that will later issue high-impact
  commands proves during channel setup that its authentication key is bound to
  an attested module so relying parties can trust subsequent privileged actions.

### Anonymous Trust Establishment

Definition: The secure channel is established based solely on attestation
Evidence, without relying on long-term stable identifiers such as PKI-based
credentials.

Protocol Requirements:

* Support for authentication frameworks where PKI is absent.

* Strict privacy preservation (e.g., encryption of the attestation payload) to
  prevent hardware identifiers from acting as persistent tracking vectors.

Instantiation Examples:

* Zero-Touch Device Onboarding: A newly manufactured device connects to a
  network without an X.509 certificate. The channel is authenticated entirely
  via the device proving its hardware genuineness (e.g., via a DICE chain),
  bootstrapping initial trust prior to standard network identity assignment.

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

## Runtime Attestation

Evidence collected at certificate issuance or during the initial secure channel establishment reflects only the Target Environment’s state at that moment. It cannot guarantee that the Target Environment remains trustworthy for the lifetime of the certificate or even for the duration of the TLS session. As a result, such static evidence is insufficient in environments where the Target Environment may change state after the connection is established and the connection is long-lived.

Runtime attestation closes this gap by enabling the Relying Party (RP) to request new attestation evidence once the TLS connection has been established, or periodically during long-lived connections if necessary.
This may be the case when the target environment has attributes that can change during the connection, affecting its trustworthiness. Such changes cannot be detected using evidence collected earlier.
For example, the evidence may include dynamic parameters such as runtime configuration flags (e.g., FIPS mode), where a device may enter or exit an approved mode, or measurements of critical system files.

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
