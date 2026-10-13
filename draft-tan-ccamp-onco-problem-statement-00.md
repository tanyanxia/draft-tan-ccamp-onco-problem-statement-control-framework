---
title: "Optical Networks and AI Computing Orchestration (ONCO) Problem Statement, Use Cases and Requirements"
abbrev: "ONCO: Problem, Use Cases, Requirements"
docname: "draft-tan-ccamp-onco-problem-statement-00"
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

Distributed artificial intelligence (AI) computing is increasingly deployed across geographically dispersed AI data centers (AIDCs) to meet the scale and performance demands of modern AI workloads. In such environments, the efficiency of distributed training and inference depends critically on tight coordination between optical transport networks and compute orchestration systems. However, today's infrastructure operates with isolated control planes: optical networks lack awareness of dynamic compute requirements, while compute schedulers have no visibility into real-time network conditions such as latency, bandwidth, or congestion. This decoupling leads to suboptimal resource utilization, degraded job performance, and inefficient scaling.

This draft presents the problem statement, outlines two representative use cases: distributed AI training and distributed AI inference, and specifies the requirements for Optical Networks and AI Computing Orchestration (ONCO). The goal is to enable bidirectional awareness, joint resource abstraction, and synchronized control across the compute-optical boundary, thereby supporting intent-driven, end-to-end provisioning of AI services over wide-area optical infrastructures.

--- middle

# Introduction

The rapid development of large-scale AI applications, particularly those involving distributed training and inference across geographically dispersed AIDCs, has exposed a critical gap in today's infrastructure: the lack of coordination between optical transport networks and compute orchestration systems. While optical networks provide the high-bandwidth, low-latency, and deterministic connectivity required for wide-area AI computing collaboration, their control planes remain largely agnostic to the dynamic, heterogeneous demands of AI workloads. Conversely, compute schedulers operate without visibility into the underlying network's real-time state, such as path latency, available bandwidth, or congestion levels.

This decoupling leads to suboptimal resource utilization, degraded job performance, and inefficient scaling of distributed AI jobs. For example, a training job may be scheduled across distant AIDCs with abundant GPU resources but poor optical connectivity, resulting in prolonged synchronization phases and significant compute efficiency loss. Similarly, latency-sensitive inference services may be routed through paths that meet compute criteria but violate network service-level objectives.

To address these challenges, ONCO enables bidirectional awareness, joint resource abstraction, and synchronized control across the compute-optical boundary. This draft describes sample usage scenarios that drive ONCO requirements and will help to identify candidate solution architectures and solutions.

## Requirements Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as shown here.

# Terminology

This document uses the following terms:

Hard-Isolated:
: A transport channel that is physically separated from other traffic at the optical layer (e.g., dedicated wavelength or time-slot), ensuring zero interference, zero jitter, and guaranteed bandwidth. Unlike soft isolation (e.g., queue-based or priority-based mechanisms), hard isolation provides deterministic performance regardless of background traffic load.

Compute Efficiency Loss:
: The degradation of compute resource utilization caused by communication bottlenecks, synchronization delays, or resource contention. In distributed AI workloads, this typically manifests as GPU idle cycles waiting for parameter synchronization or data transfer, resulting in reduced effective FLOPS and prolonged job completion times.

Key-Value (KV) Cache:
: A data structure used in autoregressive language model inference that stores the key and value tensors from the attention mechanism for previously processed tokens. The KV Cache enables efficient generation by avoiding recomputation of attention states, but can grow to hundreds of gigabytes for long context windows and large batch sizes, necessitating efficient migration mechanisms across distributed inference nodes.

Time-to-First-Token (TTFT):
: The latency measured from the moment an inference request is submitted to the moment the first output token is generated. TTFT is a critical user-perceived latency metric for interactive AI services, particularly in conversational and real-time applications.

Time-Per-Output-Token (TPOT):
: The average time required to generate each subsequent output token after the first token. TPOT characterizes the sustained generation throughput of an inference service and directly impacts the responsiveness of interactive AI applications.

# Problem Statement

~~~ ascii-art
+------------------------------------------+
|        AI Compute Service Request        |
|    (e.g., Distributed Training Job)      |
+--------------------+---------------------+
                     |
          +----------v----------+
          |                     |
          v                     v
+------------------+   +---------------------+
| Compute          |   | Optical Network     |
| Scheduler        |   | Controller          |
+------------------+   +---------------------+
          |                        |
          |                        |
          v                        v
+------------------+            +-----------------+
|     AIDC-A       |<--Low-BW-->|     AIDC-B      |
| GPU: High        |(e.g., 100G)| GPU: High       |
| Load: Low        |            | Load: Low       |
+--------+---------+            +---------+-------+
        |                                |
        |                                |
        |    High-BW (e.g., 400G)        |
        +<------------------------------>+
        |                                |
        |                                |
        |    High-BW (e.g., 400G)        |
        +<------------------------------>+
        |                                |
+--------v---------+                     |
|     AIDC-C       |<--------------------+
| GPU: Low         |
| Load: High       |
+------------------+
~~~
{: title="Suboptimal Resource Allocation under Decoupled Control" artwork-name="problem.txt"}

## Isolated Control and Management

The primary challenge lies in the management and control isolation between the computing domain (e.g., AI training clusters, cloud pools) and the optical transport network. This separation creates a "chasm" that prevents holistic resource optimization.

The optical network control plane operates without awareness of the real-time characteristics and requirements of the compute jobs it carries. Crucially, the physical layer properties of the optical infrastructure directly determine the feasibility and efficiency of AI workloads.

The physical distance between geographically distributed AIDCs imposes a fundamental propagation delay (roughly 5 microseconds per kilometer). For large-scale distributed training, this inherent latency directly impacts the time of global synchronization operations (e.g., All-Reduce). Furthermore, traditional packet-switched networks may introduce variable jitter, queuing delays, and potential packet loss. These can lead to frequent retransmissions and high synchronization penalties, causing massive GPU idle cycles and severely degrading compute efficiency.

The optical underlay possesses unique physical-layer capabilities, such as wavelength or time-slot switching, which can provision hard-isolated and deterministic circuits. These circuits eliminate jitter and guarantee a stable, latency-bounded channel. This is a critical differentiator for synchronizing massive GPU clusters across long distances. Without this physical-layer awareness, the control plane cannot leverage these unique deterministic features to support high-performance AI workloads.

Conversely, the compute orchestration layer (e.g., Kubernetes schedulers, AI job managers) makes resource allocation and job placement decisions based primarily on local compute and storage metrics (e.g., GPU availability, memory). It lacks visibility into the underlying network's physical state, including the end-to-end propagation delay and whether the available optical paths can provide the required deterministic performance. This results in compute jobs being scheduled across locations with poor physical connectivity (e.g., long-distance links with excessive latency or shared channels lacking isolation). Such scheduling causes significant communication bottlenecks and degraded overall job performance (i.e., compute efficiency loss).

This bidirectional lack of awareness creates a fundamental mismatch. While the optical network possesses the physical-layer capabilities (deterministic latency, hard-isolation) to act as the ideal transport for wide-area AI workloads, these capabilities are not exposed to or understood by the compute domain. Consequently, neither domain can adapt to the needs of the other, severely limiting the potential of wide-area collaborative computing.

## Independent Resource Efficiency Evaluation

Compounding the control isolation is the absence of a unified framework for evaluating the joint efficiency of compute and network resources.

Today, network performance is evaluated using traditional metrics like bandwidth utilization, latency, and packet loss. Compute performance is assessed through metrics like FLOPS (Floating Point Operations Per Second), job completion time, and resource utilization (CPU/GPU/memory). These evaluation systems each operate independently.

There is no standardized method to quantify the combined cost and benefit of a joint compute-and-network resource allocation decision. For instance, as shown in Figure 1, it is difficult to answer whether allocating a more powerful but distant GPU (with higher network latency) is more efficient than a less powerful but local one for a specific AI job.

This lack of a common evaluation language prevents the development of truly optimal co-scheduling algorithms that balance compute power against network quality.

## Limitations of Existing Computing-Aware Frameworks

Existing frameworks, such as Computing-Aware Traffic Steering (CATS){{I-D.ietf-cats-framework}} {{I-D.ietf-cats-usecases-requirements}}, have initiated efforts to integrate computing metrics into network routing decisions. However, these approaches are primarily designed for traffic steering at the network layer and fall short of the orchestration requirements for AI workloads over optical infrastructures. Specifically, two fundamental limitations exist.

First, existing frameworks cannot provision optical links on demand. CATS focuses on selecting an optimal path among already existing and established network routes. It lacks the capability to interact with the optical control plane to create new physical or logical links. This limitation becomes critical in cross-regional AI scheduling scenarios.

For example, a distributed training job spanning Beijing and Shanghai (1000 km apart) requires a high-bandwidth, low-latency, and deterministic optical channel to synchronize model parameters efficiently. The physical propagation delay over 1000 kilometers is a hard constraint, but the more unpredictable issue lies in packet jitter and queuing delays introduced by traditional IP-based forwarding. CATS-based traffic steering over existing shared pipes cannot guarantee the required deterministic performance, leaving the training job vulnerable to synchronization stalls and severe GPU utilization degradation.

In contrast, the optical physical layer can establish hard-isolated channels through dedicated wavelength or time-slot circuits (e.g., OXC or fgOTN). These channels offer deterministic latency, zero jitter, and guaranteed bandwidth, which are essential for efficient all-reduce operations over long distances. The existing framework cannot trigger this on-demand optical provisioning, which represents a fundamental gap.

Second, existing frameworks do not support full lifecycle orchestration of computing resources. They are generally limited to steering service requests to service instances that are already available. They cannot perform explicit reservation, locking, and subsequent release of high-performance resources (like GPU clusters).

In the Beijing-Shanghai scenario, effective orchestration cannot merely "steer" traffic. It must first jointly reserve compute resources at both AIDCs and then trigger the optical layer to provision a end-to-end deterministic optical containers to deliver the training job. Without this joint capability, both networks and computing remain in isolated silos, which prevents the consistent, high-performance execution of distributed AI workloads.

# Use Cases

The growing scale and distribution of artificial intelligence workloads have created new demands on wide-area optical infrastructure, particularly in scenarios that span multiple artificial intelligence data centers (AIDCs). Two representative use cases illustrate the need for tighter integration between optical transport networks and compute orchestration systems.

## Distributed AI Training

In distributed training of large language models (LLMs), state-of-the-art techniques like 3D parallelism (data, pipeline, and tensor parallelism) coordinate thousands of GPUs across geographically dispersed AIDCs. The dominant synchronization primitive, Ring All-Reduce, is highly sensitive to the quality of inter-AIDC connections.

Consider a training job spanning Beijing and Shanghai (1000 km). The Round-Trip Time (RTT) is physically bound to roughly 10ms by the speed of light in fiber. However, standard RoCEv2 (RDMA over Converged Ethernet) networks rely on Priority Flow Control (PFC) to prevent packet loss. In a wide-area context, PFC can cause "head-of-line" blocking and deadlocks, leading to severe jitter and occasional large latency spikes. These unpredictable fluctuations cause the All-Reduce operation to stall. This results in massive GPU idle cycles and renders the training efficiency of GPUs, especially those using high-end H100 or Blackwell architectures.

Existing computing-aware routing frameworks (such as CATS) are designed for "steering" traffic over pre-existing paths and cannot eliminate this jitter. The core issue is the buffer-based queuing inherent to IP forwarding. To achieve deterministic performance, the optical physical layer is a superior choice.

An Optical Cross-Connect (OXC) can provision an ultra-low-latency, bufferless, and jitter-free wavelength circuit between two AIDCs. For a 1000km link, this provides a deterministic, hard-isolated channel with a fixed propagation delay (e.g., 5ms one-way) and zero queuing latency. This deterministic channel ensures that intra-iteration synchronization (All-Reduce) completes within a predictable time window, significantly improving GPU utilization and training stability.

## Distributed AI Inference

Distributed AI inference, specifically for LLMs, introduces unique data dispersal requirements. The Key-Value (KV) Cache stores attention states for autoregressive generation. For long context windows and large batch sizes, the KV Cache can reach hundreds of gigabytes.

In a cross-regional inference scenario (e.g., an incoming request is processed by a central AIDC but must be offloaded to a regional edge node for sustained conversation), the KV Cache may need to be relocated or remotely accessed across the wide-area network.

If the KV Cache is migrated over a standard IP network, congestion and unpredictable buffer delays can significantly degrade the Time-to-First-Token (TTFT) and Time-Per-Output-Token (TPOT) metrics. For real-time conversational AI, even a single retransmission or queuing event can cause a noticeable stall or abrupt increase in latency, breaking the user experience. This is a critical bottleneck for implementing distributed inference services across data centers.

To enable seamless KV Cache migration, the transport network must provide a high-bandwidth, deterministic, and low-latency channel. An optical path (e.g., an fgOTN timeslot or OXC wavelength) can be dynamically provisioned to form a dedicated pipe specifically for the cache migration. This guarantees that the KV Cache transfer is not impacted by other data plane traffic, ensuring predictable TTFT and TPOT.

Furthermore, the optical path's guaranteed latency allows the inference scheduler to accurately predict the cost of remote KV Cache access or migration. This enables optimal placement decisions (e.g., co-locating the prompt processing and KV Cache affinity with the optical path quality).

# Requirements

Based on the problem statements and use cases described above, the following functional and performance requirements are necessary to support Optical Networks and AI Computing Orchestration (ONCO). These requirements are essential for enabling joint, high-performance scheduling of AI workloads across wide-area optical infrastructures.

## Integrated Control and Management Architecture

An integrated control architecture is required to break down the management silos. This architecture must facilitate bidirectional fusion between the compute and network control planes.

Specifically, the architecture must support the following *performance requirements*:

* **Rapid Service Provisioning:** The response time for establishing a cross-domain, joint compute-and-optical-path service (e.g., initiating a new training job across two AIDCs) should be compressed from the current hour-level down to **sub-minute levels**.

* **Fast Resource Adjustment:** The control plane must support dynamic reconfiguration of optical bandwidth and compute resources in response to workload changes (e.g., scaling a training job).

* **Deterministic Performance Monitoring:** The architecture must expose real-time, end-to-end optical performance metrics to the joint orchestrator. This includes, but is not limited to, **Link OSNR (Optical Signal-to-Noise Ratio)**, **pre-FEC BER (Bit Error Rate)**, and **propagation delay**.

* **Resilience and Protection:** The architecture must support rapid optical path protection switching in the event of OSNR degradation or fiber failure. This high availability is crucial for long-duration training jobs, preventing massive compute wastage due to a single optical fault.

## Unified Abstraction of Computing and Network Resources

A common language is needed for both domains to understand each other's capabilities and state. This requires a unified resource abstraction model.

The optical network's heterogeneous resources must be abstracted into a model that can be understood by the joint orchestrator. This model must expose detailed and quantifiable parameters including:

* **Physical Layer Health:** Explicit parameters such as **Path**. These are necessary to assess the viability of a long-haul optical circuit for high-throughput, low-error-rate AI traffic.

* **Deterministic QoS:** Guaranteed **Available Bandwidth** (in Gbps), **One-way Minimum Latency**, and **Jitter Bounds**.

* **Topology Constraints:** Abstract topology information must be exposed to support multi-domain path computation.

Similarly, heterogeneous compute resources (CPU, GPU, memory, storage) must be abstracted into a model that conveys their real-time capabilities and state to the joint orchestrator. This model must include:

* **Compute Capabilities:** Metrics such as **GPU FLOPS** (FP16/FP8), **VRAM Size** (in GB), and **Available Memory Bandwidth** (in GB/s).

* **Job Characteristics:** The joint orchestrator must be able to indicate the **expected bandwidth requirement**, **latency sensitivity**, and **duration target** of a specific AI job (e.g., training vs. inference).

This unified, detailed abstraction is the foundation for any credible joint decision-making process.

## Joint Orchestration of Computing and Network Resources

Building on the integrated architecture and unified abstraction, a joint orchestration mechanism is required to make end-to-end resource allocation decisions. This mechanism must:

* **Optimize for a Joint Objective:** The joint orchestrator must be capable of simultaneously evaluating a multi-dimensional objective function composed of both compute performance metrics and network performance metrics. It should be able to trade off compute power against network quality to find the overall service optimum.

* **Compute-Aware Optical Provisioning:** The joint orchestrator must be able to initiate an **end-to-end deterministic optical circuit setup** (e.g., OXC cross-connect, fgOTN cross-connect) to support a specific AI workload.

* **Network-Aware Compute Resource Locking:** The joint orchestrator must support the **reservation and locking** of compute resources (e.g., GPU clusters) that take into account the **underlying optical path**.

* **Cross-Domain Lifecycle Management:** The joint orchestrator must manage the end-to-end lifecycle of a ONCO service (e.g., from intent to teardown), including joint resource reservation, path setup, dynamic adjustment, and coordinated release across both compute and optical domains.

# IANA Considerations {#IANA}

This document makes no request for IANA action.

# Security Considerations {#Security}

TBD

--- back

<!-- raw RFCXML references: preserve exact BibXML URLs, draft revisions, and groups -->
{::nomarkdown}
<references xmlns:xi="http://www.w3.org/2001/XInclude">
      <name>References</name>
      <references>
        <name>Normative References</name>
        <xi:include href="https://bib.ietf.org/public/rfc/bibxml/reference.RFC.2119.xml" />
        <xi:include href="https://bib.ietf.org/public/rfc/bibxml/reference.RFC.8174.xml" />
      </references>

      <references>
        <name>Informative References</name>
        <xi:include href="https://bib.ietf.org/public/rfc/bibxml3/reference.I-D.draft-ietf-cats-framework-16.xml" />
        <xi:include href="https://bib.ietf.org/public/rfc/bibxml3/reference.I-D.draft-ietf-cats-usecases-requirements-08.xml" />
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
