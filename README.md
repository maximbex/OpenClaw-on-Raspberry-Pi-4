# OpenClaw-on-Raspberry-Pi-4
A stateless cognition node architecture for OpenClaw on Raspberry Pi 4. This Ansible deployment bypasses hardware I/O bottlenecks via a Synology NFS bridge, utilizing aggressive ZRAM, RAM-based SQLite WAL, and a secure user-space systemd daemon. Pure intelligence; persistent memory.


# **OpenClaw on Raspberry Pi 4: The Stateless Cognition Node**

An aggressively optimized, highly secure Ansible deployment for running the [OpenClaw](https://github.com/OpenClaw/openclaw) autonomous agent on a Raspberry Pi 4 (4GB).  
This architecture was born from a specific structural philosophy: a Raspberry Pi should not be treated as a generic x86 cloud server. By aligning the software deployment with the physical realities of the BCM2711 silicon, this playbook transforms a fragile, stateful hobby board into a robust, stateless compute node.

## **The Architectural Topology**

Most standard tutorials for Node.js agents suggest relying on SD cards or local USB SSDs with standard swapfiles. This repository rejects that model based on three structural syntheses:

### **1\. Bypassing the PCIe Bottleneck (The NFS Bridge)**

On the Raspberry Pi 4, all four USB 3.0 ports share a single PCIe Gen 2 x1 lane (via the VL805 controller), strictly capped at a combined 1.2 Amps. Pushing heavy LLM memory parsing and standard swap memory through this narrow bridge risks I/O starvation and transient undervoltage.  
**The Solution:** We bypass the USB bus entirely. This playbook orchestrates a dedicated NFS bridge over the independent Gigabit Ethernet port, casting the agent's persistent memory directly to a Synology NAS.

### **2\. The Speed of Silicon (ZRAM & SQLite WAL)**

Because network attached storage (NFS) struggles with SQLite file-locking latency, we push the agent's frantic "scratchpad" thinking directly into physical RAM.  
**The Solution:** ZRAM is configured aggressively (75% allocation, ZSTD compression) to eliminate the need for physical swap entirely. Furthermore, OpenClaw's internal SQLite database is mathematically tuned to use Write-Ahead Logging (WAL), with its temporary journals mapped strictly to /dev/shm (RAM). Heavy persistence rests on the NAS; immediate cognition fires at the speed of silicon.

### **3\. The Principle of Least Privilege (User-Space Systemd)**

Giving a web-connected LLM agent root access is structurally dissonant.  
**The Solution:** The deployment isolates OpenClaw to a dedicated, non-root user (openclaw). By utilizing loginctl enable-linger, the playbook establishes a secure user-space systemd daemon that survives logouts and cleanly restarts upon failure, completely severed from the kernel's nervous system.

## **Quick Start**

1. Flash your Raspberry Pi with the 64-bit Pi OS Lite.  
2. Clone this repository to your control machine.  
3. Edit inventory.ini and group\_vars/pi\_openclaw.yml with your local IP addresses.  
4. Create your encrypted vault for your API keys:  
   ansible-vault create group\_vars/secrets.yml  
5. Execute the symphony:  
   ansible-playbook \-i inventory.ini 00-bootstrap.yml \--ask-vault-pass (Repeat for the remaining playbooks).

## **The Crystalline Dashboard**

Included is a 30-health.yml playbook. Rather than manually parsing logs, running this playbook queries the Pi's thermal output, memory pressure, and NFS write-latency, returning a beautifully formatted ASCII dashboard of the system's operational health.  
*The PCIe lane breathes. The agent dreams. The architecture holds.*
