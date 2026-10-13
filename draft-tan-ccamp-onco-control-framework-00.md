---
title: "A Control Framework for Optical Networks and AI Computing Orchestration (ONCO)"
abbrev: "ONCO Control Framework"
docname: "draft-tan-ccamp-onco-control-framework-00"
category: "info"
submissiontype: "IETF"
ipr: "trust200902"
v: 3
lang: "en"
pi: {sortrefs: false}
date: "2026"
area: "General"
workgroup: "ccamp"
author:
  - name: "Yanxia Tan"
    initials: "Y"
    surname: "Tan"
    org: "China Unicom"
    email: "tanyx11@chinaunicom.cn"
  - name: "Zheng Han"
    initials: "Z"
    surname: "Han"
    org: "Beijing University of Posts and Telecommunications"
    email: "2025010255@bupt.cn"
  - name: "Xiao Li "
    initials: "X"
    surname: "Li"
    org: "Huawei Technologies"
    email: "lixiao33@huawei.com"
  - name: "Sanwei Yang"
    initials: "S"
    surname: "Yang"
    org: "ZTE Corporation"
    email: "yang.sanwei1@zte.com.cn"
  - name: "Xing Zhao"
    initials: "X"
    surname: "Zhao"
    org: "CAICT"
    email: "zhaoxing@caict.ac.cn"
--- abstract

This document defines the control framework for Optical Networks and AI Computing Orchestration (ONCO). The framework is designed to achieve synergistic management of optical network (e.g., fgOTN, OXC) and computing resources for high-performance AI workloads. It specifies a multi-stakeholder service model, a layered architecture across management, control, and data planes, and a set of functional components. The ONCO framework supports both centralized and distributed orchestration models to enable proactive resource reservation and deterministic optical circuit provisioning.

--- middle

# Introduction {#introduction}

The rapid evolution of Artificial Intelligence (AI), characterized by large-scale model training and distributed inference, has shifted the requirements for wide-area infrastructure from simple connectivity to deterministic optical-compute orchestration. High-performance AI workloads, particularly compute-to-compute interactions among geographically distributed AI Data Centers (AIDCs), demand deterministic low latency, stable high bandwidth, bounded jitter, strict isolation, and explicit resource guarantees that go beyond traditional best-effort transport. The key technical distinction of the Optical Networks and AI Computing Orchestration (ONCO) framework is a shift from an IP-overlay traffic steering model to an optical-underlay resource orchestration model.

The Computing-Aware Traffic Steering (CATS) framework {{I-D.ietf-cats-framework}} provides an overlay architecture to dynamic selection of service instances based on joint network and computing awareness. While CATS excels at steering service-specific traffic toward existing instances over available network pipes, it is primarily a traffic-triggered mechanism that operates on a "selection" and "awareness" basis. In a traditional IP or IP-overlay environment, the underlay transport resources may still be statistically multiplexed and affected by queueing, congestion, packet-layer path changes, and contention with other services. As a result, an overlay mechanism alone is not sufficient to guarantee the fixed latency, low jitter, hard isolation, and congestion-loss-free behavior required by emerging AI scenarios such as geographically distributed training. These scenarios require a more proactive approach centered on admission control, advance resource reservation, and deterministic path provisioning in the underlay network.

ONCO therefore extends the core principles of compute-aware networking into the optical transport domain and uses fine-grained Optical Transport Network (fgOTN) and Optical Cross-Connect (OXC) technologies as the underlying optical network capabilities. fgOTN supports small-granularity bandwidth allocation, deterministic transport containers, predictable latency, low jitter, and service-level hard isolation, making it suitable for reserving optical resources for AI flows with different bandwidth and timing requirements. OXC enables optical-layer cross-connect and circuit-level path provisioning, reducing packet-layer processing and supporting stable high-capacity optical paths. The combination of fgOTN and OXC provides a deterministic optical underlay that can support fixed or predictable delay, bounded jitter, and elimination of congestion-induced packet loss under normal fault-free operation, which demonstrates its suitability and advantages over a purely traditional IP-based transport approach for AI optical-compute services.

Unlike pure traffic-steering approaches that react to packets arriving at the ingress, ONCO focuses on the end-to-end orchestration of resource lifecycles before traffic is admitted. It enables the reservation of compute resources and the provisioning of new deterministic optical circuits triggered by service intents or API requests prior to traffic flow. Through coordination among the AI Service Orchestrator (ASO), Joint Optical-Compute Orchestrator (JOCO), Compute Power Controller (CPC), and Provisioning Network Controller (PNC), ONCO can align compute placement with optical path setup, resource locking, SLA assurance, and resource release across the full service lifecycle.

The rest of this document is organized as follows:

* Section 3 defines the service model for AI computing over optical networks, clarifying the roles and responsibilities of Customers, Service Providers, Network Providers, and Computing Power Providers.
* Section 4 describes the ONCO layered architecture and details the specific functional entities across the management, control, and data planes.
* Section 5 specifies the interaction procedures between these functional components to achieve joint resource reservation and provisioning in both centralized and distributed orchestration scenarios.

## Requirements Language {#requirements-language}

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as shown here.

# Terminology {#terminology}

The terms defined in the CATS framework {{I-D.ietf-cats-framework}} and the ACTN framework {{RFC8453}} are used in this document. To support the joint orchestration of AI computing and deterministic optical networks, the following terms are introduced or redefined. Detailed architectural descriptions of these functional components are provided in Section 4.2.

{: newline="true"}
AI Service Orchestrator (ASO)
: A management-plane entity that accepts high-level AI service intents and translates them into structured resource demands.

Joint Optical-Compute Orchestrator (JOCO)
: The centralized control-plane entity responsible for end-to-end joint optical-compute orchestration.

Edge JOCO
: A distributed control-plane entity deployed at an edge optical network node, capable of autonomous remote AIDC selection and optical path provisioning based on the extended joint traffic engineering database (TED) that contains computing power information.

Compute Power Controller (CPC)
: A control-plane agent that monitors and locks resources within the AI compute pool.

Provisioning Network Controller (PNC)
: The ACTN transport-domain controller reused by ONCO for the optical underlay. The PNC monitors optical network states, abstracts optical topology, and executes deterministic optical circuit provisioning.

ASO-JOCO Interface (AJI)
: The northbound management interface between the ASO and the JOCO for conveying high-level AI service intents and resource demands. It can be viewed as an enhanced ACTN CMI for AI service intents.

JOCO-CPC Interface (JCI)
: The southbound interface between the JOCO and the CPC for monitoring and reserving of compute resources.

MDSC-PNC Interface (MPI)
: The ACTN interface defined in {{RFC8453}} between the JOCO, acting as an enhanced MDSC, and the PNC for monitoring and provisioning of optical networks. ONCO does not define a separate optical-controller interface.

Optical-Compute Gateway (OC-GW)
: A protocol mediation gateway deployed at the AIDC edge, facilitating the exchange of compute metrics between the compute domain and the optical network. It can translate compute metrics into network-layer TE attributes and trigger distributed routing or signaling mechanisms; extended OSPF-TE {{RFC3630}} and extended RSVP-TE {{RFC3209}} are possible examples.

# Service Model for AI Computing over Optical Network

The deployment of wide-area AI services over optical infrastructure involves multiple stakeholders, each playing a distinct role in the end-to-end service delivery chain. To clarify responsibilities and interactions, this document defines a service model comprising the Customer, Service Provider, Network Provider, and Computing Power Provider.

## Customer

The Customer is the end user or enterprise that consumes AI capabilities. Three primary service patterns are observed:

In AI training, the customer delegates the training of large-scale AI models to service providers, typically specifying performance, scale, and data privacy requirements.

In AI inference, the customer leases computing resources to deploy and operate inference models, often serving downstream internet users with real-time or batch inference services.

## Service Provider

The Service Provider acts as the business orchestrator, interfacing directly with the Customer to translate high-level service intents—such as SLAs, geographic constraints, or performance targets—into concrete resource demands. It coordinates with both the Network Provider and the Computing Power Provider to fulfill these demands, and is responsible for service lifecycle management, billing, and customer support.

## Network Provider

The Network Provider operates and manages the underlying optical transport infrastructure. It delivers high-bandwidth, low-latency, and deterministic connectivity services, including inter-AIDC backbone links and user-to-AIDC dedicated access circuits. The Network Provider exposes network capabilities—such as available bandwidth, path latency, and reliability—through standardized control interfaces to enable coordinated service provisioning.

## Computing Power Provider

The Computing Power Provider owns and operates one or more Artificial Intelligence Data Centers (AIDCs). It offers compute, memory, and accelerator resources (e.g., GPUs, TPUs) for AI training and inference workloads. The Computing Power Provider reports real-time resource availability and performance metrics to the Service Provider and supports dynamic task placement and scaling based on orchestration instructions.

# ONCO Control and Management Architecture

The ONCO framework defines a layered architecture to achieve synergistic orchestration of AI computing and optical transport. This chapter details the functional entities and their roles in both centralized and distributed orchestration models.

## Architecture Overview {#arch-overview}

The ONCO architecture consists of three functional planes: the Management Plane for intent translation, the Control Plane for joint resource orchestration and metric distribution, and the Data Plane for deterministic forwarding. The centralized JOCO model and the distributed Edge JOCO model are alternative deployment modes and are not used simultaneously for the same ONCO service. Unlike CATS, ONCO does not assume that traffic is steered only toward already deployed service instances. Instead, ONCO coordinates AI compute resource reservation in AIDCs with deterministic optical path provisioning. The two deployment modes are illustrated in Figure 1 and Figure 2.

~~~ ascii-art
   +-------------------------------------------------------------+
   | ONCO Management Plane                                       |
   |        +---------------------------------------+            |
   |        | AI Service Orchestrator (Enhanced CNC)|            |
   |        +---------------------+-----------------+            |
   +------------------------------|------------------------------+
                                  |AJI
   +------------------------------|------------------------------+
   | ONCO Control Plane           |                              |
   |                  +-----------v-----------+                  |
   |                  | JOCO (Enhanced MDSC)  |                  |
   |                  +-----------+-----------+                  |
   |               +----------+   |   +----------+               |
   |               | JCI          | MPI          | JCI           |
   |               |              |              |               |
   |           +---v---+       +--v--+       +---v---+           |
   |           | CPC A |       | PNC |       | CPC Z |           |
   |           +---+---+       +--+--+       +---+---+           |
   +---------------|--------------|--------------|---------------+
                   |              |              |
                   |              |              |              
   +---------------|--------------|--------------|---------------+
   |ONCO Data Plane|              |              |               |
   |               |              |              |               |
   |           +---v----+         |         +----v---+           |
   |           | AIDC A |         |         | AIDC Z |           |
   |           +---+----+         |         +----+---+           |
   |               |              |              |               |
   |      +--------|--------------v--------------|--------+      |
   |      | +------v-----------+     +-----------v------+ |      |
   |      | |    C-TC A        |     |       C-TC Z     | |      |
   |      | |------------------|     |------------------| |      |
   |      | | CATS-Forwarder A |     | CATS-Forwarder Z | |      |
   |      | +------+-----------+     +-----------+------+ |      |
   |      |        |                             |        |      |
   |      |        |      +----------------+     |        |      |
   |      |        +====> | Deterministic  |<====+        |      |
   |      |               | Optical Path   |              |      |
   |      |               | over fgOTN/OXC |              |      |
   |      |               +----------------+              |      |
   |      +-----------------------------------------------+      |
   +-------------------------------------------------------------+
~~~
{: title="Centralized ONCO Architecture" #fig-centralized-architecture artwork-name="problem.txt"}

~~~ ascii-art
   +-----------------------------------------------------------------------+
   |ONCO Management Plane                                                  |
   |        +---------------------------------------+                      |
   |        | AI Service Orchestrator (Enhanced CNC)|                      |
   |        +--------------------+------------------+                      |
   +-----------------------------|-----------------------------------------+
                                 |
   +-----------------------------|-----------------------------------------+
   |ONCO Control Plane           |                                         |
   |               +-------------v--------+                                |
   |               | JOCO (Enhanced MDSC) |                                |
   |               +-------------+--------+                                |
   |                             |                                         |
   |      +-------+  NETCONF +---v---+      +-------+ NETCONF +-------+    |
   |      |OC-GW A|<---------| CPC A |      | CPC Z |-------->|OC-GW Z|    |
   |      +---+---+          +---+---+      +---+---+         +---+---+    |
   |          |                  |              |                 |        |
   |      +---v---+              |              |             +---v---+    |
   |      | Edge  |              |              |             | Edge  |    |
   |      |JOCO A |<====== Routing/Signaling Mechanisms =====>|JOCO Z |    |
   |      +---+---+              | (e.g., ASON/GMPLS)         +---+---+    |
   |          |                  |              |                 |        |
   +----------|------------------|--------------|-----------------|--------+
              |                  |              |                 |
              v                  v              v                 v
   +----------|------------------|--------------|-----------------|--------+
   |          |              +---v----+    +----v---+             |        |
   |  +-------v--------+     | AIDC A |    | AIDC Z |  +----------v-----+  |
   |  |     C-TC A     |<----+--------+    +--------+->|     C-TC Z     |  |
   |  +----------------+                               +----------------+  |
   |  |CATS-Forwarder A|                               |CATS-Forwarder Z|  |
   |  +--------+-------+                               +---------+------+  |
   |             |         +----------------------------+        |         |
   |             +========>| Deterministic Optical Path |<=======+         |
   |                       |       over fgOTN/OXC       |                  |
   | ONCO Data Plane       +----------------------------+                  |
   +-----------------------------------------------------------------------+
~~~
{: title="Distributed ONCO Architecture" #fig-distributed-architecture}

## Functional Component Descriptions {#comp-desc}

### Management Plane Entities {#comp-mgmt}

{: newline="true"}
AI Service Orchestrator
: Description: Responsible for interfacing with customers to receive high-level AI service intents (e.g., "geographically distributed training with 10ms sync latency"). It translates these intents into specific resource requirements—such as compute capacity, memory footprint, and deterministic network bandwidth. It also handles service lifecycle management, including billing, SLA enforcement, and user authentication. It does not manage physical resources directly but instead communicates abstracted demands to the control plane via the AJI.

  Relationship to CATS: Acts as the high-level service management entity.

  Relationship to ACTN: This component conceptually maps to the Customer Network Controller (CNC) defined in {{RFC8453}}. However, instead of merely passing traditional Virtual Network Service (VNS) requirements, it translates and communicates overarching AI service intents (encompassing both computing demands and deterministic network demands) to the control plane via the AJI. Accordingly, the AJI can be regarded as an enhanced CNC-MDSC Interface (CMI) for AI-oriented optical-compute orchestration.

### Control Plane Entities {#comp-control}

The control plane supports two independent orchestration models (Centralized and Distributed). The ONCO architecture uses ONCO-specific names for new or enhanced optical-compute orchestration functions and uses ACTN and CATS terminology where ACTN or CATS functions are directly reused. These entities describe logical functions; depending on implementation and deployment, a function may be realized as a standalone component or hosted by, co-located with, or located adjacent to another network or compute-domain component.

{: newline="true"}
Joint Optical-Compute Orchestrator (JOCO)
: Description: The centralized decision-making engine for the ONCO framework. It manages the entire lifecycle of resources, performing joint reservation of compute resources and initiating the provisioning of new deterministic optical circuits. It coordinates across both the Compute Domain and the Optical Domain to establish synergistic end-to-end services.

  Relationship to CATS: Inherited from the CATS Path Selector (C-PS). However, while the standard C-PS focuses on path selection and traffic steering for existing routes, the JOCO proactively reserves compute resources and provisions new deterministic optical circuits.

  Relationship to ACTN: This component acts as an Enhanced Multi-Domain Service Coordinator (Enhanced MDSC). Unlike the standard ACTN MDSC that solely coordinates network domains, this enhanced entity natively incorporates computing power. It coordinates across both the Compute Domain and the Optical Domain to establish synergistic end-to-end services.

Provisioning Network Controller (PNC)
: Description: Monitors the optical underlay in the centralized orchestration model. It tracks parameters such as wavelength availability, fgOTN timeslot utilization, and physical layer latency, and executes the setup of optical channels based on orchestration decisions. The PNC may be deployed as a standalone optical controller or hosted in, or located adjacent to, an operator control platform or the nearest AIDC.

  ONCO reuses the ACTN PNC for the optical transport domain.

  The PNC abstracts the optical topology and exposes it to the JOCO via the MPI, and receives provisioning commands to configure optical nodes.

Compute Power Controller (CPC)
: Description: Monitors the AI compute pool, collecting real-time telemetry such as GPU utilization, VRAM availability and task queue status. It executes task placement and compute resource locking commands within the Artificial Intelligence Data Center (AIDC). The CPC may be deployed as a standalone compute control component or hosted in, or located adjacent to, the AIDC whose compute resources it controls. It abstracts non-network compute resources (e.g., GPU clusters) and exposes them to the JOCO via the dedicated JCI.

  Relationship to CATS: Inherited from the CATS Service Metric Agent (C-SMA). It is enhanced from simple service state awareness to supporting active, stateful compute resource reservation and locking.

  Relationship to ACTN: This component conceptually maps to the Provisioning Network Controller (PNC) defined in {{RFC8453}}. However, rather than managing network elements, this entity introduces the novel concept of a Compute Provisioning Controller. Architecturally, its interface with the JOCO holds an equivalent hierarchical status to the MPI. However, a strict domain separation is maintained: while the optical network domain is controlled via the MPI, the compute domain is governed through the dedicated JCI.

Edge JOCO
: Description: An independent, distributed control-plane entity deployed at an edge optical network node. It may be hosted by, or co-located with, the same edge optical network node that supports the C-TC and CATS-Forwarder functions, while remaining logically separate from those data-plane functions. It performs rapid resource reservation and optical path provisioning based on locally collected metrics.

  For any specific ONCO service, the Edge JOCO and the centralized JOCO orchestration functions are mutually exclusive. This avoids having two independent orchestration authorities issue resource reservation, path computation, or signaling decisions for the same service. The deployment choice determines whether the service is controlled by a centralized orchestration loop or by a distributed orchestration loop.

  In the distributed model, the JOCO does not perform the centralized joint resource computation or optical provisioning role. It only relays the AI service resource demand from the AI Service Orchestrator to the local CPC, which then drives the distributed control chain.

  Relationship to CATS: Derived from the distributed deployment model of the CATS C-PS.

Optical-Compute Gateway (OC-GW)
: Description: A protocol mediation gateway deployed at the AIDC edge in the distributed orchestration model. It retrieves AI compute metrics from the CPC, translates them into network-layer TE attributes, and makes them available to the distributed optical control plane through routing, signaling, controller-mediated, or other resource advertisement mechanisms. Extended OSPF-TE Opaque LSAs and extended RSVP-TE signaling are possible examples for carrying metric advertisements or service requirements; they are not mandatory protocol choices for ONCO.

  Relationship to CATS: A new component specific to the ONCO distributed orchestration model to bridge compute metrics into the optical control plane.

### Data Plane Entities {#comp-data}

The data plane focuses on the high-speed, deterministic delivery of AI traffic according to control-plane decisions. The definitions of data plane entities are extended from {{I-D.zhao-cats-otn-applicability}} to support hard-isolation optical transport.

{: newline="true"}
CATS-Forwarder
: Description: Functions as the entry (Ingress) or exit (Egress) point for deterministic optical containers at an edge optical network node. In this framework, it plays a role similar to a Provider Edge (PE) node in an optical transport network, where service traffic is admitted into, or released from, the provider-controlled optical underlay. Unlike a packet-only edge function, the ONCO CATS-Forwarder relies on fgOTN and OXC capabilities to bind service flows to deterministic optical resources, including fine-grained time slots, ODUk containers, wavelengths, or optical cross-connect paths. Based on forwarding instructions from the JOCO or Edge JOCO, the Ingress CATS-Forwarder encapsulates and maps client AI signals into hard-isolation optical pipes (e.g., ODUk or fgOTN timeslots). Conversely, the Egress CATS-Forwarder decapsulates and de-maps the signals back to their original client format for delivery to the AIDC.

  Relationship to CATS: Extends the standard CATS-Forwarder from traffic forwarding over existing paths to optical-edge adaptation and provisioning support. By enabling mapping to fgOTN containers and OXC-based optical paths, it provides the hard isolation, predictable latency, bounded jitter, and high-capacity deterministic transport that cannot be guaranteed by an IP-overlay forwarding function alone.

CATS Traffic Classifier (C-TC)
: Description: A data-plane functional entity typically hosted in, or located adjacent to, the Ingress CATS-Forwarder at an edge optical network node. It is responsible for identifying incoming AI service traffic through physical ports, VLAN tags, or specific Service IDs (CS-ID). It ensures that identified flows are correctly mapped and encapsulated into the appropriate pre-provisioned optical containers (e.g., fgOTN) as directed by the JOCO or Edge JOCO.

  Relationship to CATS: Aligns with the standard CATS C-TC but is strictly tied to optical container mapping rules in the OTN data plane.

# Applicable scenarios of Distributed and Centralized Architectures {#applicability-of-architectures}

Editor's note: This section describes the scenarios where distributed and centralized orchestration architectures are most applicable.

# ONCO Operational Workflows {#operational-workflows}

This section details the end-to-end operational workflows for AI service deployment over optical networks. It covers the full lifecycle—resource provisioning, utilization, flexible adjustment, and release—across both centralized and distributed orchestration models. In both models, the user's high-level intent (e.g., AI training or inference model deployment) is ingested via the AI Service Orchestrator.

## Centralized Orchestration Workflow {#workflow-centralized}

In the centralized model, the JOCO is responsible for all metric collection, global optimization, and top-down command dispatch for both compute and network resources.

~~~ ascii-art
+----+  +---------+  +-------------+  +--------------+  +-------------+
|User|  | AI Orch |  |     JOCO    |  |     CPC      |  |     PNC     |
+----+  +---------+  +-------------+  +--------------+  +-------------+
  |          |              |                |                |
  |[Phase 1: Resource Provisioning]          |                |
  |          |              |                |                |
  |1. Intent |              |                |                |
  |--------->| 2. Translate |                |                |
  |          |---Demands--->| 3. Periodic Metric Collection   |
  |          |              |<---Compute-----|                |
  |          |              |<-------Network------------------|
  |          |              |                |                |
  |          |              |---+ 4. Calculate Joint          |
  |          |              |<--+ Resource Allocation         |
  |          |              |                |                |
  |          |              | 5a. Provision Optical Path      |
  |          |              |-------------------------------->|
  |          |              | 5b. Lock Compute Resources      |
  |  Deployed|    Deployed  |--------------->|                |
  |<---------|<-------------|                |                |
  |          |              |                |                |
  |[Phase 2: Utilization and Flexible Adjustment]             |
  |          |              |                |                |
  | 6. User -driven         | 7. State-driven|                |
  |--------->|------------->|<---Alert-------|                |
  |          |              |<-------Alert--------------------|
  |          |              |-Re-calculate-->|                |
  |          |              |-Re-calculate------------------->|
  |          |              |                |                |
  |[Phase 3: Resource Release]               |                |
  |          |              |                |                |
  | 8. Terminate            |                |                |
  |--------->|--Release---->|--Release------>|                |  
  |          |              |--------Teardown---------------->|
~~~
{: title="Centralized Orchestration Sequence" #fig-centralized-flow}

{: newline="true"}
Phase 1: Resource Provisioning
: The user submits an AI task intent to the AI Service Orchestrator (Step 1), which translates it into specific compute and network demands and forwards them to the JOCO (Step 2). Utilizing its global view gathered via both the dedicated JCI for compute metrics and the MPI for network states (Step 3), the JOCO calculates the optimal joint allocation for both compute and optical resources (Step 4). It then dispatches commands to the PNC to provision the deterministic optical circuit (Step 5a) and the CPC to lock the compute resources (Step 5b).

Phase 2: Utilization and Flexible Adjustment
: During task execution, adjustments can be triggered by two sources. User-driven adjustments occur when the customer updates the SLA (e.g., scaling up compute resources) via the AI Service Orchestrator (Step 6). State-driven adjustments occur when the CPC or PNC detects threshold crossings (e.g., GPU memory exhaustion or optical degradation) and alerts the JOCO (Step 7). In both cases, the JOCO re-computes and dispatches update instructions to dynamically scale resources without interrupting the service.

Phase 3: Resource Release
: Upon completion of the AI task, the AI Service Orchestrator triggers a termination signal. The JOCO releases the compute instances via the CPC and tears down the optical connections via the PNC (Step 8).

## Distributed Orchestration Workflow {#workflow-distributed}

In the distributed model, orchestration is decentralized. Compute and service availability is advertised across the distributed control plane so that edge nodes can maintain an abstract view of optical and compute resource states. The AI Service Orchestrator translates an AI service intent into resource demands and sends them to the JOCO. In this model, the JOCO does not perform centralized joint resource computation; it relays the demand to the local CPC, which then drives the distributed optical-compute control chain. The edge optical nodes in the sequence below are PE nodes that host the Edge JOCO control-plane function and support the C-TC and CATS-Forwarder data-plane functions. The detailed protocol procedures used to advertise resources, provision paths, or release reservations are outside the scope of this framework and may be specified in separate protocol extension documents.

~~~ ascii-art
+-----+   +-------+  +--------+     +--------+   +-------+    +-----+
|CPC A|   |OC-GW A|  |  PE-A  |     |  PE-Z  |   |OC-GW Z|    |CPC Z|
+-----+   +-------+  +--------+     +--------+   +-------+    +-----+
   |          |          |              |             |          |
   |          |     [Phase 1: Resource Awareness]     |          |
   |--Metric->|          |              |             |<-Metric--|
   |          |--Metric->|              |<---Metric---|          |
   |          |          |<--Res view-->|             |          |
   |          |          |              |             |          |
   |    [Phase 2: Endpoint Selection and Link Establishment]     |
   |--Demand->|          |              |             |          |
   |          |-Forward->|              |             |          |
   |          |          |---+ selects  |             |          |
   |          |          |<--+ endpoint |             |          |
   |<---ACK---|<---ACK---|              |             |          |
   |          |------------------Path---------------->|          |
   |          |<---------------ACK--------------------|          |
   |          |          |              |             |          |
   |      [Phase 3: Service Initialization and Execution]        |
   |          |------Initialization Service---------->|          |
   |          |          |              |             |--Init--->|
   |          |          |              |             |<---ACK---|
   |          |<---------------ACK--------------------|          |
   |          |          |              |             |-Service->|
   |          |          |              |             |<-Result--|
   |          |<--------------Result------------------|          |
   |<-Result--|          |              |             |          |
   |          |          |              |             |          |
   |         [Phase 4: Adjustment, Recovery, or Release]         |          
   |--Update->|<--Alert--|              |             |          |
   |          |<-----------------Alert----------------|<--Alert--|
   |          |--Adjust->|              |             |          |
   |          |          |---+ Re-calculate           |          |
   |          |          |<--+          |             |          |
   |          |<-Result--|              |             |          |
   |          |------------Adjust Path--------------->|          |
   |          |          |              |             |--Adjust->|
   |          |          |              |             |<---ACK---|
   |<---ACK---|<-----------------ACK------------------|          |
   |          |          |              |             |          |
   |-Release->|-------------------------------------->|          |
   |          |          |              |             |-Release->|
   |          |<-----------------ACK------------------|<---ACK---|
   |          |----------------Teardown-------------->|          |
   |<---ACK---|                                       |          |
~~~
{: title="Distributed Orchestration Sequence" #fig-distributed-flow}

{: newline="true"}
Phase 1: Resource Awareness
: The distributed orchestration process relies on an abstract resource view that reflects both compute availability and optical network state. CPC A and CPC Z collect compute pool status from their respective AIDCs and make this information available to PE-A and PE-Z through the corresponding OC-GWs. PE-A and PE-Z are edge optical network nodes that host the Edge JOCO control-plane function and support the C-TC and CATS-Forwarder data-plane functions. Resource advertisements are exchanged across the distributed optical control plane so that the Edge JOCO functions can maintain an abstract resource view for subsequent endpoint selection and resource coordination. The advertisement function may be supported by distributed resource advertisement mechanisms, for example extensions to OSPF-TE, but the ONCO framework does not mandate a specific protocol realization.

Phase 2: Endpoint Selection and Link Establishment
: After the AI service demand has been relayed to the local CPC, CPC A checks local compute feasibility and, when local compute resources are insufficient or remote compute resources are explicitly required, delegates the remote service request to OC-GW A. OC-GW A forwards the demand to the local PE-A edge node, where the Edge JOCO function selects a suitable remote compute/service endpoint, such as the PE-Z edge node and its associated AIDC, based on the abstract resource view. After the endpoint is selected, PE-A establishes deterministic connectivity toward PE-Z through the distributed optical-compute control chain. The deterministic connectivity establishment function may be supported by distributed signaling, controller-based provisioning, or other path provisioning mechanisms; the sequence shown here only describes the architectural function.

Phase 3: Service Initialization and Execution
: After the deterministic link has been established, the distributed control chain triggers service initialization on the selected remote side. The remote side prepares the service runtime, initializes the requested AI service instance, and confirms readiness back to the requesting side. Once the service initialization is acknowledged, the C-TC identifies the relevant AI service flow, and the CATS-Forwarder maps the classified flow into the selected deterministic optical resource. Inference or other AI service results are returned through the distributed control chain to the AI Service Orchestrator as needed by the service lifecycle.

Phase 4: Adjustment, Recovery, or Release
: During service execution, the distributed control chain may handle service adjustment, recovery, or release. A top-down service update may modify the AI service requirement while retaining the existing compute and optical connectivity resources, or may trigger endpoint reselection and connectivity adjustment. A bottom-up condition, such as compute load change, resource conflict, or optical path failure, may trigger recovery procedures including endpoint reselection, establishment of an alternative deterministic optical connection, and release or teardown of the affected optical connection. Upon service suspension, the AI service may be stopped while associated compute and optical connectivity resources are retained according to policy. Upon service deletion, the same distributed control chain releases both compute and optical connectivity resources according to the requested service policy.

# IANA Considerations {#IANA}

This document makes no request for IANA action.

# Security Considerations {#Security}

The ONCO framework introduces a deeply integrated orchestration mechanism across compute and optical network domains. Because this framework handles high-value AI workloads, sensitive compute metrics, and core optical transport resources, securing the control, management, and data planes is critical. The security considerations are divided into system-level security and interface-level security.

## System Security {#system-security}

System security focuses on the robustness, isolation, and availability of the end-to-end ONCO architecture.

* **Resource Isolation and Multi-tenancy:** AI training and inference tasks from different customers MUST be strictly isolated. In the data plane, the optical underlay SHOULD utilize hard-isolation technologies (e.g., fgOTN timeslots, ODUk) to prevent cross-talk and side-channel attacks between tenants. In the compute domain, robust virtualization or bare-metal isolation is required.

* **Denial of Service (DoS) Protection:** Malicious actors might attempt to exhaust valuable compute (GPU) or deterministic optical resources by flooding the AI Service Orchestrator with fake intent requests. The system SHOULD implement strict rate-limiting, resource quotas, and admission control at the AI Service Orchestrator and JOCO levels.

* **Topology and Compute State Confidentiality:** The CPC collects detailed compute pool metrics (e.g., GPU utilization, task queues). If exposed, this data could reveal a Computing Power Provider's business scale or operational status. Abstraction and topology-hiding mechanisms (similar to ACTN Grey/Black topologies) MUST be applied before exposing intra-domain states to a multi-domain JOCO.

* **High Availability and Single Point of Failure:** In the centralized orchestration model, the JOCO acts as a single decision-making engine. It SHOULD be deployed with high availability, state synchronization, and geographic redundancy to prevent catastrophic failures caused by targeted attacks.

## Interface Security {#interface-security}

The ONCO architecture relies on multiple standardized interfaces to bridge the management, control, and functional agents. Data transmitted over these interfaces MUST be protected against eavesdropping, spoofing, and man-in-the-middle (MitM) attacks.

* **Northbound Interface (AJI):** The interface between the AI Service Orchestrator (ASO) and the JOCO handles customer SLA and AI intents. It MUST support robust Authentication, Authorization, and Accounting (AAA). Secure transport protocols (e.g., TLS/HTTPS) and Role-Based Access Control (RBAC) are REQUIRED to ensure that only authorized entities can request or modify resources.

* **Southbound Interfaces (MPI, JCI & NETCONF):** The JOCO communicates with the PNC via the MPI and with the CPC via the dedicated JCI. Similarly, in the distributed model, the local CPC communicates with the OC-GW via NETCONF. These interfaces transmit precise telemetry and provisioning directives. They MUST utilize mutual authentication (e.g., mTLS) and data encryption to prevent the malicious injection of false compute metrics or unauthorized optical cross-connect teardowns.

* **Control Plane Protocols:** In the distributed workflow, resource advertisement and deterministic connectivity establishment may be supported by extensions to routing or signaling protocols, for example OSPF-TE and RSVP-TE. When such protocols are used, standard routing and signaling protocol security mechanisms MUST be enabled. For example, cryptographic authentication for OSPFv2 {{RFC5709}} and RSVP {{RFC2747}} SHOULD be deployed to prevent rogue nodes from injecting false compute/network states or tearing down deterministic paths maliciously.

--- back

<!-- raw RFCXML references: preserve exact BibXML URLs, draft revisions, and groups -->
{::nomarkdown}
<references xmlns:xi="http://www.w3.org/2001/XInclude">
      <name>References</name>
      <references>
        <name>Normative References</name>
        
        <xi:include href="https://bib.ietf.org/public/rfc/bibxml/reference.RFC.2119.xml" />
        <xi:include href="https://bib.ietf.org/public/rfc/bibxml/reference.RFC.8174.xml" />
        <!-- The recommended and simplest way to include a well known reference -->
      </references>

      <references>
        <name>Informative References</name>
        <!-- CATS Framework: 核心基础 -->
        <xi:include href="https://bib.ietf.org/public/rfc/bibxml3/reference.I-D.ietf-cats-framework.xml" />
        
        <!-- CATS OTN Applicability: 光网络确定性转发基础 -->
        <xi:include href="https://bib.ietf.org/public/rfc/bibxml3/reference.I-D.zhao-cats-otn-applicability.xml" />

        <!-- RSVP-TE: 用于光路预留与信令扩展参考 -->
        <xi:include href="https://bib.ietf.org/public/rfc/bibxml/reference.RFC.3209.xml" />

        <!-- OSPF-TE: 用于 OC-GW 指标分发扩展参考 -->
        <xi:include href="https://bib.ietf.org/public/rfc/bibxml/reference.RFC.3630.xml" />
        <xi:include href="https://bib.ietf.org/public/rfc/bibxml/reference.RFC.8453.xml" />
        <xi:include href="https://bib.ietf.org/public/rfc/bibxml/reference.RFC.2747.xml" />
        <xi:include href="https://bib.ietf.org/public/rfc/bibxml/reference.RFC.5709.xml" />
      </references>

    </references>
{:/nomarkdown}

<!-- raw RFCXML Contributors section: keep contacts nested under their section -->
{::nomarkdown}
<section anchor="Contributors" numbered="false">
      <name>Contributors</name>
      <t>The following people contributed significantly to this document:</t>
      <contact fullname="Wei Wang" initials="W" surname="Wang">
        <organization>Beijing University of Posts and Telecommunications</organization>
        <address>
          <email>weiw@bupt.edu.cn</email>
        </address>
      </contact>
      <contact fullname="Jie Zhang" initials="J" surname="Zhang">
        <organization>Beijing University of Posts and Telecommunications</organization>
        <address>
          <email>jie.zhang@bupt.edu.cn</email>
        </address>
      </contact>
      <contact fullname="Yongli Zhao" initials="Y" surname="Zhao">
        <organization>Beijing University of Posts and Telecommunications</organization>
        <address>
          <email>yonglizhao@bupt.edu.cn</email>
        </address>
      </contact>
      <contact fullname="Qiaojun Hu" initials="Q" surname="Hu">
        <organization>Beijing University of Posts and Telecommunications</organization>
        <address>
          <email>qiaoj475@bupt.edu.cn</email>
        </address>
      </contact>
      <contact fullname="Yanlei Zheng" initials="Y" surname="Zheng">
        <organization>China Unicom</organization>
        <address>
          <email>zhengyanlei@chinaunicom.cn</email>
        </address>
      </contact>
    </section>
{:/nomarkdown}
