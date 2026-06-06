# GAIA: Distributed Artificial Intelligence Architecture
## High-Performance Llama.cpp Implementation on Overclocked Pi 5

### Overview
A high-performance, 4-node Raspberry Pi 5 computing cluster engineered for distributed LLM inference, optimized local token generation, and real-time model horizontal sharding.

The project represents a deep-dive into maximizing ARM-based silicon, moving away from the overhead of container orchestration toward a high-performance **Bare Metal** architecture for the inference engine. This design ensures the cluster can handle 20B parameter models with minimal token-generation latency.

GAIA is a modular, agent-based framework designed to bridge the gap between static Large Language Models (LLMs) and dynamic, data-driven environments. Built with a focus on Retrieval-Augmented Generation (RAG) and Agentic Workflows, GAIA enables the automation of complex reasoning tasks by integrating with local and cloud-based data sources.

## 1. Architectural Overview

Project GAIA splits model inference workloads across four physical nodes over a high-throughput, Power-over-Ethernet (PoE) switched backplane. To eliminate container virtualization overhead and ensure direct, unthrottled access to hardware registers and memory controllers, the primary text generation pipelines run directly on host bare metal.

### Hardware Topology
* **Compute Nodes:** 4× Raspberry Pi 5 (8GB LPDDR5X-4267)
* **Silicon Optimization:** Stable 2.6 GHz CPU overclock; active undervolting profile for thermal mitigation.
* **Power Delivery & Cooling:** Waveshare PoE+ Hat configuration drawing from a managed **UniFi Switch Ultra 60W**.
* **Storage Architecture (The Vault):** Decoupled, low-latency NFSv4 share offloading heavy model tensors and parameter weights from local SD/NVMe storage to dedicated network storage.

### Subnet Map & Role Matrix
| Hostname | Node IP | Architectural Role | Execution Layer |
| :--- | :--- | :--- | :--- |
| **`io`** | `192.168.86.201` | Swarm Manager / Main API Anchor | Standalone Docker (Control Plane) + Bare-Metal |
| **`europa`** | `192.168.86.202` | Swarm Worker / Compute Thread 01 | Bare-Metal Inference Plane |
| **`ganymede`** | `192.168.86.203` | Swarm Worker / Compute Thread 02 | Bare-Metal Inference Plane |
| **`callisto`** | `192.168.86.204` | Swarm Worker / Compute Thread 03 | Bare-Metal Inference Plane |

---

## 2. The Core Production Stack

### Bare-Metal Inference Plane (`llama.cpp`)
The text-generation pipeline leverages native `llama.cpp` implementations built from source using optimized compilation flags (`-march=native -O3`). 
* Each system executes an individual shard runner bound via a unified `systemd` unit file (`llama-inference.service`).
* Nodes expose native endpoints on network port `50052`, allowing upstream orchestrators or application layers to issue tensor compilation requests directly to the raw hardware.

### Containerized Monitoring & Management Sidecars
While the heavy mathematical lifting is isolated to bare metal, infrastructure management uses a highly optimized, lightweight Docker Swarm configuration monitored via a centralized pane of glass:
* **Portainer Server & Agent Mesh:** Portainer Server runs on `io` and links to standalone global agents running on host ports `9001` across the cluster over an attachable `portainer_agent_network` overlay backplane. This configuration allows complete cluster management from a single visual dashboard without interfering with bare-metal memory structures.
* **Telemetry Reporting:** Lightweight **Glances** containers run locally on each engine node to pipe real-time thermals, RAM saturation, and CPU utilization metrics to upstream dashboards.

---

## 3. Engineering Post-Mortem: The Exo Orchestration Experiment

### Objective
In an effort to achieve true p2p sharding and transparently slice massive models (e.g., Llama-3-70B) across the 32GB collective pool of LPDDR5X cluster memory, we attempted to implement **Exo** (`exo-explore`), an open-source distributed inference framework designed to link heterogeneous devices into a single virtual GPU/NPU.

### Technical Roadblocks Encountered
1. **Dynamic Topology Failures over Managed Fabrics:** Exo relies heavily on automated device discovery and dynamic peer-to-peer connection pooling. Over the UniFi switched layer, internal security definitions and multicast configurations caused intermittent peer drops, causing inference queries to fracture mid-token.
2. **Memory Over-Allocation Panics:** Because Exo dynamically handles tensor allocation based on reported system profiles, it consistently misjudged the explicit memory ceilings of the Pi 5 platform under heavy CPU overclocks, initiating aggressive Kernel Out-Of-Memory (OOM) kills that brought down the host SSH daemons.
3. **Virtualization Interface Overhead:** Running Exo inside automated execution environments introduced severe packet translation latency. When sharding a model's layers across four nodes, the time-to-first-token (TTFT) degraded exponentially due to container network interface (CNI) bridging loops.

### The Solution: Re-Baselining to Explicit Bare Metal
To overcome these challenges, the cluster was pivoted away from dynamic P2P orchestration frameworks to an architecture defined by **Predictable, Explicit Isolation**:
* **Exo Eviction:** Exo was completely stripped from the production runtimes.
* **Deterministic Sharding:** Replaced dynamic routing with fixed, predictable model quantization profiles manually distributed across the network filesystem.
* **Network Simplification:** Erased multi-layered Docker Swarm ingress routing and dynamic virtual bridges. The current configuration relies purely on static network sockets on host port `50052`.
* **Stability Gains:** This shift eliminated OOM engine crashes entirely, stabilized CPU core temperatures under full load, and established a dependable baseline for multi-user inference.

---

## 4. Future Roadmap
* [ ] **Distributed RAG Ingestion Layer:** Integrating an isolated, containerized Vector Database sidecar (Qdrant/Chroma) using persistent disk mappings without altering the bare-metal execution parameters.
* [ ] **Optimized Context Window Expansion:** Fine-tuning flash attention alternatives compiled specifically for the ARM Neon vector engine architecture.

---

Maintained by Scienz_Guy | 2026
