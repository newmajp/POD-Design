# Architectural Design Decisions & Deployment Blueprint
## On-Premise Hyperscale Pods for Big Data Footprints

This document outlines the core engineering guidelines, network topologies, storage profiles, component bills of materials (BOM), and lifecycle management strategies required to construct and run modular, high-capacity big data pods in on-premise infrastructure.

---

## 1. Core Architectural Strategy & Pillars

### 1.1 Disaggregated Shared-Everything (DASE)
By decoupling the high-performance compute engines from the central data storage matrix using a **VAST Data DASE architecture**, the pod achieves linear, cloud-like scalability. Storage capacity and processing power scale independently without mutual bottlenecks.

### 1.2 Multi-Fabric Network Segregation
To ensure deterministic execution times for large database transactions and heavy analytics shuffles, network communication is divided into two physically separate line fabrics:
*   **The Converged Compute & DB Fabric:** Manages inter-node application shuffles, Kubernetes pod-to-pod links, and low-latency database traffic. Traffic separation and security are maintained using strict **IEEE 802.1Q VLAN isolation** and Quality of Service (QoS) tiering.
*   **The Resilient Storage Fabric:** An independent, isolated **NVMe-over-Fabrics (NVMe/TCP)** mesh network that directly links compute workers to the distributed VAST engine.

### 1.3 High-Availability Blast Radius Mitigation
*   **Cross-Cabinet Switch Layering:** The primary **Compute Leaf Switches** are installed diagonally across Rack 1 and Rack 2, while the **Storage Leaf Switches** are co-located as an **MLAG / vPC pair** in Rack 3.
*   **Segmented Active/Active Database:** The active/active database infrastructure (**PostgreSQL + Patroni + MaxScale**) is split evenly across all three cabinets (1 DB host per rack) to safeguard against local power distribution unit (PDU) failures.
*   **Stateless Storage Elements:** VAST CNodes are distributed across all three racks (1 per cabinet) to guarantee storage availability during localized rack maintenance windows.

---

## 2. Pod Footprint Options

### Option A: 15 kW Mid-Density Footprint
*   **Sustained Continuous Load Limit:** 12.0 kW per rack (following the standard 80% facility headroom rule).
*   **Total Compute Density:** 31 Compute Nodes (11 in Rack 1, 13 in Rack 2, 7 in Rack 3).
*   **Switch Selection:** 36-Port Leaf Switches.
*   **Cabling Profile:** Intra-rack Direct Attach Copper (DAC) with standard inter-rack Active Optical Cables (AOC).

### Option B: 30 kW High-Density Footprint
*   **Sustained Continuous Load Limit:** 24.0 kW per rack.
*   **Total Compute Density:** 67 Compute Nodes (25 in Rack 1, 31 in Rack 2, 11 in Rack 3).
*   **Switch Selection:** 64-Port High-Density Leaf Switches.
*   **Cabling Profile:** High-density AOC trunk bundles passing through overhead structured fiber raceways. Requires active containment or **Rear Door Heat Exchangers (RDHx)**.

### Option C: 7 kW Low-Density Footprint
*   **Sustained Continuous Load Limit:** 5.6 kW per rack.
*   **Total Compute Density:** 13 Compute Nodes (5 in Rack 1, 5 in Rack 2, 3 in Rack 3).
*   **Switch Selection:** 36-Port Leaf Switches.
*   **Cabling Profile:** Optimized primarily with long DAC runs; minimal AOC trunks required. Designed for standard corporate server rooms.

---

## 3. Storage Architecture: VAST Data Engine

The storage fabric delivers a raw capacity of **20 Petabytes (20PB)** built from two fundamental building blocks:

### 3.1 VAST CNodes (Compute Nodes)
*   **Quantity:** 3 Nodes, split 1 per rack across the pod.
*   **Function:** Stateless protocol handlers that accept incoming S3 API requests, native NFS-over-RDMA shares, or VAST CSI calls. They abstract data formatting operations from physical flash chips.

### 3.2 VAST DNodes (Data Nodes)
*   **Quantity:** 2 Nodes, co-located within Rack 3.
*   **Function:** Highly dense physical storage enclosures populated with Enterprise QLC NVMe flash drives for bulk storage capacity, paired with low-latency Storage Class Memory (SCM) modules to protect data writes and handle metadata indexing.

### 3.3 Container Storage Interface (CSI) Integration
Kubernetes workloads orchestrate storage attachment via two distinct plugins:
*   **VAST Block CSI Driver (`ReadWriteOnce` / RWO):** Used for low-latency databases or scratch storage. Maps block devices natively using automated Virtual IP (VIP) assignments to balance paths evenly across all active CNodes.
*   **VAST Filesystem CSI Driver (`ReadWriteMany` / RWX):** Empowers concurrent container engines to process shared directories and parquet files simultaneously during scale-out big data query paths.

---

## 4. The Database Tier: Active/Active PostgreSQL

To balance writes across the distributed infrastructure safely, the database layer leverages a decoupled architecture on bare-metal servers:

```
                  [ Compute Workloads (K8S / Hadoop) ]
                                    │
                       (Virtual Floating Master IP)
                                    │
                         ┌──────────┴──────────┐
                         ▼                     ▼
                   [ MaxScale 1 ]        [ MaxScale 2 ]
                   (Proxy/Router)        (Proxy/Router)
                         │                     │
         ┌───────────────┼─────────────────────┴───────────────┐
         ▼               ▼                                     ▼
  [ Postgres DB 1 ] [ Postgres DB 2 ]                   [ Postgres DB 3 ]
   (Active Master)  (Sync Standby)                        (Sync Standby)
      (Rack 1)         (Rack 2)                              (Rack 3)
```

1.  **Patroni & Consensus Clustering:** Patroni manages the PostgreSQL runtime state using a distributed consensus tool (such as etcd or Consul) over the network. If the active master crashes or a rack drops power, Patroni initiates an automated failover sequence in less than 30 seconds.
2.  **MaxScale Proxies:** Run as redundant ingress points across the network fabric. They monitor Patroni's runtime configurations, handle connection state pools, split read operations from write paths, and present a single floating IP address to application clients.

---

## 5. Network Interconnect & Cabling Matrix

### 5.1 Physical Medium Selection Guidelines
*   **Direct Attach Copper (DAC):** Deployed for all connections inside the same rack. DAC cables minimize port power draw and lower network transport latencies.
*   **Active Optical Cables (AOC):** Mandated for all inter-rack cross-connections passing through overhead tray lines. Thin optical lines maximize airflow clearance and maintain signal clarity over structural distances.

### 5.2 Network Optimization Settings (Runbook)
To achieve full line-rate performance across the pod fabric, ensure the following parameters are applied:
*   **Jumbo Frames:** Configure standard MTU sizes to **9000** across all compute network adapters, storage interfaces, and switch leaf ports to decrease CPU parsing overhead during large file transfers.
*   **RoCEv2 Enforcements:** Enable **PFC (Priority Flow Control)** and **ECN (Explicit Congestion Notification)** on all compute leaf switches to guarantee zero packet drops for high-performance remote direct memory access (RDMA) traffic.

---

## 6. The Convergence Lifecycle Roadmap: Hadoop to Kubernetes

This section outlines how to systematically transition existing legacy Hadoop architectures onto a unified, cloud-native Kubernetes environment inside the pod.

### Phase 1: Unlinking Local Disks & Decoupling Storage
*   **Action:** Reconfigure legacy Hadoop MapReduce and Spark processing routines to utilize the `s3a://` or VAST-native file connectors.
*   **Outcome:** Direct data pipelines away from individual local server drives and stream files into the central 20PB VAST Data layer instead. This step turns your data workers into stateless compute systems.

### Phase 2: Introducing Cloud-Native Schedulers
*   **Action:** Deploy the open-source **Spark-on-K8S Operator** and containerized **Trino/Presto** search instances inside the Kubernetes pool.
*   **Outcome:** Direct new processing tasks onto the container cluster, bypassing legacy YARN schedulers without modifying primary data storage locations.

### Phase 3: Hardware Re-Provisioning Lifecycle
*   **Action:** As old Hadoop processing needs fall, use automated lifecycle workflows to systematically drain, flash, and re-image servers:

```
 ┌──────────────────────┐      ┌──────────────────────┐      ┌──────────────────────┐
 │ Legacy Hadoop Node   │ ───► │ Evacuate Data Tasks  │ ───► │ Trigger PXE Script   │
 │ (YARN Worker Daemon) │      │ (Drain Workloads)    │      │ (Wipe OS Drives)     │
 └──────────────────────┘      └──────────────────────┘      └──────────────────────┘
                                                                        │
                                                                        ▼
 ┌──────────────────────┐      ┌──────────────────────┐      ┌──────────────────────┐
 │ Joins Active Pod Pool│ ◄─── │ Install Kubelet Agent│ ◄─── │ Flash Base OS Image  │
 │ (Available K8S Node) │      │ (Register to K8S)    │      │ (Ubuntu / RHEL Minimal)
 └──────────────────────┘      └──────────────────────┘      └──────────────────────┘
```

---

## 7. Disaster Recovery (DR) Blueprint

To guarantee resilience against data center outages, the pod employs an asynchronous cross-site mirroring strategy:

### 7.1 VAST Native Snapshot Streaming
*   The primary VAST platform at Site A generates regular, non-disruptive storage snapshots.
*   Modified blocks stream across the WAN path to an identical 20PB VAST environment at Site B. 
*   This native storage-level replication preserves data reduction savings (deduplication and compression) over the WAN, minimizing external network costs.

### 7.2 Cross-Site Database Mirroring
*   The Patroni instance at Site B runs as an isolated **Standby Cluster**.
*   It ingests continuous PostgreSQL Write-Ahead Logs (WAL) via asynchronous streaming from the active Site A primary master node.
*   This approach isolates the production database fabric from inter-site network latency spikes.

### 7.3 Unified Failover Checklist
In the event of a catastrophic outage at Site A, administrators execute the following failover runbook:
1.  **Redirect Compute Path:** Update global traffic routing layers (GTM/DNS) to point ingress services toward Site B's Kubernetes entry nodes.
2.  **Elevate Database Cluster:** Signal Site B's Patroni environment to break replication lock and elevate the standby database instance to full writable Primary status.
3.  **Remount Persistent Volumes:** Trigger the VAST CSI Operator at Site B to mount the replicated storage snapshots, allowing standby containers to quickly pick up data tracking lines.
