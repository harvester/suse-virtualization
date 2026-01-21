---
title: "Harvester Networking Strategy: Tuning VM-to-VM TCP Performance"
author: Harvester Engineering Team
tags: [Harvester, Networking, Performance, KubeVirt, Linux Tuning]
---

# Harvester Networking Strategy: Tuning VM-to-VM TCP Performance

In the world of Hyperconverged Infrastructure (HCI), "it works out of the box" is our primary directive. However, for platform architects pushing high-performance workloads—such as distributed databases, AI/ML pipelines, or high-frequency trading—"working" is not the same as "optimized."

A common scenario we encounter in the field involves users deploying Harvester on high-performance 25GbE or 100GbE bare metal nodes, only to find that single-stream TCP performance between Virtual Machines (VMs) plateaus far below the wire speed (often stuck at 3-5 Gbps).

Why does this happen? The default TCP stacks in most Linux distributions are tuned for the Wide Area Network (WAN)—prioritizing stability and memory conservation over raw speed. In a Harvester cluster, where VMs sit millimeters apart on a high-speed backplane, these defaults starve your connections.

Drawing on strategies from [ESnet’s Linux Tuning](https://fasterdata.es.net/host-tuning/linux/) and our internal architecture, this guide details a **three-layer strategy** to unlock the full bandwidth of your Harvester cluster: The **Physical Node**, the **KubeVirt Abstraction**, and the **Guest VM**.

---

## Layer 1: The Harvester Node (Host Tuning)

Harvester runs on an immutable operating system (SLE Micro-based). Unlike a traditional Linux server where you might casually edit `/etc/sysctl.conf`, Harvester requires specific configuration methods to ensure changes survive reboots and upgrades.

### 1.1 The Jumbo Frames Strategy (MTU 9000)
Standard Ethernet frames (1500 bytes) create excessive CPU overhead at 25Gbps+ speeds due to the sheer number of interrupts required to process packet headers.

**The Harvester Way:**
We strongly advise *against* changing the MTU of the default `mgmt` network, as this can disrupt the control plane and CNI overlays. Instead, leverage Harvester's **ClusterNetwork** feature to isolate high-speed traffic.

1.  Navigate to **Networks > ClusterNetworks**.
2.  Create a dedicated network (e.g., `storage-data-net`) bound to your high-speed physical interfaces.
3.  Set the **MTU** to `9000` in the Uplink settings.
4.  Attach your data-heavy VM networks to this new ClusterNetwork.

### 1.2 Persistent Sysctl Configuration
To widen the TCP windows on the host, you must apply settings via the Harvester Configuration file (`config-map` or installation config).

**Recommended `os.sysctls` block:**
```yaml
os:
  sysctls:
    # Max Receive/Send Buffer (64MB)
    "net.core.rmem_max": "67108864"
    "net.core.wmem_max": "67108864"
    # Increase backlog for high connection rates
    "net.core.netdev_max_backlog": "30000"
    # TCP Autotuning: Min / Default / Max
    "net.ipv4.tcp_rmem": "4096 87380 33554432"
    "net.ipv4.tcp_wmem": "4096 65536 33554432"

## Layer 2: The Virtualization Layer (KubeVirt & VirtIO)

Even with a tuned host, the virtual network interface (vNIC) creates a bottleneck. Harvester utilizes KubeVirt, which relies on `virtio` drivers.

### 2.1 Enable VirtIO Multiqueue
By default, a vNIC maps to a single queue, meaning a *single* vCPU core on the host handles all network interrupts for that VM. If that one core hits 100% usage, your bandwidth caps out, regardless of your 100GbE link.

**The Fix:**
Enable "Multiqueue" to allow the VM to spread network packet processing across multiple vCPUs.

1.  Edit your Virtual Machine YAML configuration.
2.  Locate the `spec` -> `template` -> `spec` -> `domain` -> `devices` section.
3.  Add `networkInterfaceMultiqueue: true`.

```yaml
spec:
  template:
    spec:
      domain:
        devices:
          networkInterfaceMultiqueue: true

## Layer 3: The Guest VM (Modern TCP Algorithms)

Finally, we must tune the Guest OS. This is where modern Linux kernel features (Kernel 4.9+) shine. We need to move beyond simple buffer sizing and utilize modern **Congestion Control** and **Queue Disciplines (Qdiscs)**.

### Deep Dive: Modern Linux Networking
To tune your Harvester VMs effectively, it helps to understand two modern advancements:

1.  **BBR vs. CUBIC (Congestion Control)**
    *   **CUBIC (Traditional):** Ramps up speed until packets are lost. In virtualized environments with deep buffers, packets sit in queues ("Bufferbloat") rather than dropping immediately. CUBIC doesn't slow down until it's too late, causing latency spikes.
    *   **BBR (Modern):** Developed by Google, BBR models the network pipe. It measures bandwidth and Round-Trip Time (RTT) to send data at exactly the rate the wire can handle. It ignores random packet loss, making it ideal for high-throughput VM links.

2.  **FQ vs. FQ_CoDel (Queue Disciplines)**
    *   **FQ (Fair Queueing):** Essential for BBR. It paces packets evenly to prevent micro-bursts.
    *   **FQ_CoDel (Controlled Delay):** Excellent for latency-sensitive workloads. It drops packets proactively if they sit in the queue too long, ensuring fresh data always arrives.

### Tuning Profiles

Depending on your workload, apply one of the following profiles inside your Guest VM.

#### Profile A: The "Throughput Monster"
*Best for: MinIO Object Storage, Longhorn Replicas, Database Backups, Log Aggregation.*

This profile maximizes window sizes and uses BBR to fill the pipe.

```bash
# 1. Expand the Max OS Buffers (Allowing windows to grow large)
sysctl -w net.core.rmem_max=67108864
sysctl -w net.core.wmem_max=67108864

# 2. Tune TCP Autotuning (Min / Default / Max)
sysctl -w net.ipv4.tcp_rmem="4096 87380 33554432"
sysctl -w net.ipv4.tcp_wmem="4096 65536 33554432"

# 3. Enable BBR and Fair Queueing
# Note: BBR requires the 'fq' qdisc to pace packets correctly
sysctl -w net.core.default_qdisc=fq
sysctl -w net.ipv4.tcp_congestion_control=bbr

# 4. Increase Backlog
sysctl -w net.core.netdev_max_backlog=30000


#### Profile B: The "Low Latency Responder"
*Best for: Redis, Memcached, API Gateways, VoIP.*

This profile prevents data from sitting in buffers. We use standard congestion control combined with CoDel to keep queues empty.

```bash
# 1. Use Moderate Buffer Sizes (Prevent Bufferbloat)
sysctl -w net.core.rmem_max=8388608
sysctl -w net.core.wmem_max=8388608

# 2. Use FQ_CoDel (Fair Queueing Controlled Delay)
sysctl -w net.core.default_qdisc=fq_codel

# 3. Stick to Standard MTU (1500)
# Jumbo frames increase serialization delay, which harms latency-sensitive apps.

## Verification: The `iperf3` Test

To verify your improvements, do not rely on `scp` or file copies (which are limited by disk I/O). Use `iperf3` between two VMs on different nodes.

**Standard Test:**
```bash
iperf3 -c <server-ip>

**Parallel Stream Test (Simulates Multiqueue benefits):**
```bash
iperf3 -c <server-ip> -P 4

## Conclusion

Performance tuning in Harvester is a holistic exercise. By enabling **Jumbo Frames** on dedicated ClusterNetworks, activating **VirtIO Multiqueue** in KubeVirt, and modernizing the **TCP Congestion Control** in your guests, you can transform your Harvester cluster from a functional compute platform into a high-performance data center backbone.
