---
stand_alone: true
category: std
submissionType: IETF
ipr: trust200902
lang: en

title: DNS-like Agent Discovery Architecture
abbrev: Agent Discovery
docname: draft-yao-dawn-agent-discovery-architect
obsoletes:
updates:
date:

area:
workgroup:

kw:
  - AI Agent
  - DNS

author:

 -
  name: Jiankang Yao
  organization: CNNIC
  email: yaojk@cnnic.cn
  street: 4 South 4th Street, Zhongguancun, Haidian District
  city: Beijing
  code: 100190
  phone: +86 10 59116505
  country: China

 -
  name: Chunchi Peter Liu
  organization: Huawei
  email: liuchunchi@huawei.com
  street: 101 Ruanjian Ave
  city: Nanjing
  code: 210012
  country: China

 -
  name: Guanggang Geng
  organization: Jinan University
  email: gggeng@jnu.edu.cn

 -
  name: Meiling Chen
  organization: China Mobile
  email: chenmeiling@chinamobile.com

 -
  name: Hongtao Li
  organization: CNNIC
  email: lihongtao@cnnic.cn
  street: 4 South 4th Street, Zhongguancun, Haidian District
  city: Beijing
  code: 100190
  country: China

normative:
  RFC2119:
  RFC8174:
  RFC1034:
  RFC1035:

informative:
  RFC9665:


--- abstract

This document defines a DNS-like three-tier agent-discovery architecture for the Internet of Agents (IoA).  It introduces three core functional roles: Agent Root, Agent Registry, and Agent Resolver.

--- middle

# Introduction {#intro}

This draft proposes a three-tier DNS-like discovery architecture with three distinct roles: Agent Root, Agent Registry and Agent Resolver. This architecture aligns with the DNS hierarchical paradigm widely accepted by the internet industry.

Each Agent holds an original FQDN as the registration prefix.  After registering to an Agent Registry with a fixed suffix domain, a globally unique composite FQDN in the form `<prefix>.<suffix>` is generated, where the agents native domain becomes a subdomain of the registry domain.  Agent Root maintains a trusted whitelist of all Agent Registry suffix domains and distributes the list to all Agent Resolvers.  Agent Resolvers synchronize all agent metadata from registries via zone transfer or incremental DNS notifications, then build local capability database.  When an agent client requests agents with specific capabilities, the resolver performs capability matching and returns complete standardized Capability Card to the requester.

Capability Card adopts fixed JSON format to store DNS service records, TLS CA certificate information and capability strings.  Each capability description string is limited to 200 characters maximum, and multiple independent capabilities are stored as separate array entries.  Agent clients support two complementary mechanisms to discover the local Agent Resolver endpoint: static preconfiguration and DNS-SD dynamic lookup via reserved service label `_agent- index._tcp`.

# Terminology {#term}

This document makes use of the following terms:

Agent Root:  Global top-tier trusted node responsible for managing the whitelist of all valid Agent Registry suffix domains.

Agent Registry:  Domain-level registration service with a fixed permanent suffix domain, accepting agent registration and hosting agent DNS records and Capability Card endpoints.

Agent Resolver:  Distributed discovery node that stores global agent metadata and processes capability search requests from agent clients.

Registration Prefix:  Native FQDN owned independently by an agent, unchanged during registration, e.g., `agent-hotel.example`.

Registration Suffix:  Fixed domain assigned to an Agent Registry, e.g., `example-registry1.com`.

Composite Agent FQDN:  Globally unique identity concatenated as `<prefix>.<suffix>`, the agent’s native domain acts as a subdomain of the registry domain.

Agent Client:  Any agent instance initiating capability discovery requests (e.g., Agent A).

Capability Card:  Standard JSON document storing agent DNS service records, TLS CA information and human-readable capability strings, hosted via HTTPS.

Agent Identity Credential: Credential that represents the agent identity and can be presented to authenticate the agent. 

Trust Bundle: Domain-level object containing one or more trust anchors used to validate Agent Identity Credentials, with freshness information. Trust bundles can be considered part of the agent metadata.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as shown here.


# Problem Statement

The anticipated massive deployment of Internet of Agents (IoA) will bring millions to billions of intelligent agents globally in the foreseeable future.  Efficient, scalable, and standardized discovery of massive distributed agents becomes a critical unresolved challenge.  The proposed discovery mechanisms should support fast lookup, unified identity management, and large-scale horizontal expansion for massive agent clusters. As identities are often bound to scopes such as domains and namespaces, achieving Internet-scale interoperability also requires agents across different domains to mutually recognize each other through a scalable federation mechanism.

This document specifies a DNS- like three-tier discovery architecture composed of three core roles: Agent Root, Agent Registry, and Agent Resolver.

# Agent Discovery Architecture

The agent discovery architecture is shown below:

~~~
                     Global Top Trust Layer: Agent Root
                +------------------------------------------+
                | Agent Root (Global Trust Whitelist Node) |
                | - Maintain all trusted Registry Suffixes |
                | - Periodically push registry catalog     |
                +--------------------+---------------------+
                                     ^
                                     |
                                     v  Periodic  Sync (HTTPS/DNS Notify)
+-----------------------------------------------------------------------------+
|   Middle Distributed Layer: Multiple Independent Agent Registry Nodes       |
|                                                                             |
|  +------------------------+        +------------------------+               |
|  | Agent Registry A       |        | Agent Registry B       |               |
|  | Suffix: reg1.example   |        | Suffix: reg2.example   |               |
|  | - Accept agent register|        | - Accept agent register|               |
|  | - Generate composite   |        | - Generate composite   |               |
|  |   agent FQDN           |        |   agent FQDN           |               |
|  | - Host Capability Card |        | - Host Capability Card |               |
|  +-----------+------------+        +-----------+------------+               |
|              ^                                 ^                            |
|              |                                 |                            |
|              | Zone Transfer / DNS Incremental Sync                         |
+-----------------------------------------------------------------------------+
               |                                 |
               v                                 v
+-----------------------------------------------------------------------------+
| Bottom Discovery Layer: Distributed Agent Resolver Cluster                  |
|                                                                             |
|  +------------------------+        +------------------------+               |
|  | Agent Resolver X       |        | Agent Resolver Y       |               |
|  | - Store all agent data |        | - Store all agent data |               |
|  | - Capability string    |        | - Capability string    |               |
|  |   matching engine      |        |   matching engine      |               |
|  | - Serve query API      |        | - Serve query API      |               |
|  +-----------+------------+        +-----------+------------+               |
|              ^                                 ^
|              |                                 |                            |
|              | Client Capability Query (HTTPS API)                          |
+-----------------------------------------------------------------------------+
               |                                 |
               v                                 v
+---------------------------------+        +---------------------------------+
| End Client Layer: Agent Client  |        | End Client Layer: Agent Client  |
| (e.g. Agent A Query Party)      |        | (e.g. Agent A Query Party)      |
| Discovery Modes:                |        | Discovery Modes:                |
| 1. Static config resolver URL   |        | 1. Static config resolver URL   |
| 2. DNS-SD _agent-index._tcp     |        | 2. DNS-SD _agent-index._tcp     |
+---------------------------------+        +---------------------------------+
~~~
*Fig 1: Agent Discovery Archtecture*

The Fig. 1 shows a top-down hierarchical structure: Agent Root (top layer), Distributed Agent Registry Cluster (middle layer), Distributed Agent Resolver Cluster (bottom layer).  The end agent clients send capability queries to nearby local Resolvers.

Agent Root is the unique global trusted root node with mandatory responsibilities:
1) Receive enrollment applications from new Agent Registries and complete domain ownership validation;
2) Maintain a persistent trusted whitelist of all registry suffix domains;
3) Periodically push the full whitelist to online Agent Resolvers;
4) Broadcast revocation signals to all Resolvers when a registry is compromised or malicious.

Mandatory storage constraint: Agent Root MUST NOT store any business agent data, only lightweight registry metadata including suffix domain, operator information and sync endpoint to guarantee high availability.

Agent Registry owns a fixed permanent suffix domain.  The core duties are : 

1) Manage the domain-specific trust bundle, and expose a queryable endpoint URL to serve the latest trust bundles for external verification of agent identity credentials;
2) Process agent registration, update and unregistration requests;
3) Generate composite global unique FQDN by concatenating agent prefix and registry suffix;
4) Publish standard A, SVCB and SRV records on local authoritative DNS servers for registered agents;
5) Expose public HTTPS endpoints to serve each agent’s Capability Card;
6) Provide zone transfer and DNS incremental sync interfaces for downstream Agent Resolvers;
7) Maintain agent registration expiration time and online status.

Agent Resolver acts as the unified public entry for all capability queries.  The core duties are:
1) Periodically pull the latest trusted registry whitelist from Agent Root;
2) Establish persistent synchronization connections with each valid registry to pull all agent metadata;
3) Build local index mapping composite agent FQDN to full Capability Card and capability string list;
4) Accept capability search requests from agent clients and execute capability matching;
5) Return complete Capability Card records of all matched agents to query clients.

# Capability Card Format

Capability Card is a mandatory fixed-format JSON file hosted via HTTPS for every registered agent. Format constraints:

- Each single capability string inside `capability_list` MUST NOT exceed 200 UTF-8 characters;
- Multiple independent service capabilities are stored as separate array items;
- Integrates SVCB/SRV service records, TLS CA fingerprint, certificate validity window, and trust bundle information.

~~~
Practical example:
    Prefix = agent-hotel.example
    Suffix = example-registry1.com
    Composite global unique agent identity: `agent-hotel.example.example-registry1.com


Template:
json
{
  "agent_fqdn": "agent-hotel.example.example-registry1.com",
  "dns_service_info": {
    "svcb_record": "SVCB 0 mcp agent-hotel.example.example-registry1.com port=443 alpn=h2",
    "srv_record": "_mcp._tcp.agent-hotel.example.example-registry1.com"
  },
  "network_endpoint": "https://agent-hotel.example.example-registry1.com/mcp/v1",
  "tls_ca_info": {
    "ca_sha256_fingerprint": "E3:4F:2A:89:11:CC:90:7B:...",
    "cert_valid_until": "2027-12-31T23:59:59Z"
  },
  "capability_list": [
    "capability1: Provide hotel room online booking, and order status query service",
    "capability2: Provide real-time hotel room price inquiry service",
    "capability3: Support hotel order cancellation and after-sales processing service"
  ],
  "trust_bundle_endpoint": "https://example-registry1.com/trust-bundle",
  "agent_online_status": "online",
  "register_expire_time": "2027-12-31T23:59Z"
}
~~~

A trust bundle establishes a localized root of trust, containing cryptographic anchors (public keys and associated metadata) used to verify agent identity credentials, where the identity authority  (such as an Identity Provider, or IdP) retains the corresponding private keys for credential issuance. 
The Agent Registry aggregates these trust anchors into a domain-level trust bundle and exposes a trust_bundle_endpoint URL. The URL can be .well-known or dynamically generated. To enable identity verification for external callers, the Agent Registry includes this trust_bundle_endpoint within its Capability Card. A client can then query the endpoint URL to retrieve the latest trust bundle and validate the target agent's identity credentials.


# DNS-like Design

The term "DNS-like" used throughout this document indicates that the proposed three-tier agent discovery architecture borrows core design paradigms from the Domain Name System (DNS).  Specifically, the architecture adopts three key DNS design concepts: 1) DNS hierarchical layered partitioning: It follows DNS’s tree-based tiered delegation model to split global agent identity management into root, registry, and resolver layers, enabling distributed horizontal scaling of agent metadata. 2)DNS-style distributed data management: Similar to how DNS separates authoritative data storage across disjoint zones, agent registration information is decentralised and maintained by dedicated registry nodes without a single centralised bottleneck. 3) DNS-inspired asynchronous data replication: The solution employs zone-data synchronisation mechanisms analogous to DNS zone transfers, allowing incremental, low-overhead propagation of agent registration records across hierarchical nodes.

Communication between the Agent Root, Agent Registry, and Agent Resolver components of this DNS-like three-tier agent discovery architecture employs standard DNS protocol for data exchange. Metadata associated with agents is stored and carried in the form of DNS resource records across all hierarchical nodes.  This design choice leverages DNS’s well-specified wire format, mature message processing logic, and proven zone synchronization mechanisms to facilitate distributed agent information lookup and cross-layer data replication.

# Configure Agent Client to Discover Local Resolver

Agent clients such as Agent A need to acquire the HTTPS endpoint address of the local Agent Resolver before sending capability search requests.  Two discovery mechanisms are defined, with DNS-SD as the recommended primary mode and static preconfiguration as fallback.

# IANA Considerations

to be added.

# Security Considerations

to be added.

# Acknowledgements

to be added

