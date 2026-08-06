## Copilot instructions for ONTAP Select documentation

### Repository overview
Product: ONTAP Select

ONTAP Select is a software-only version of ONTAP that runs as a virtual machine on hypervisor hosts (VMware ESXi or KVM), delivering enterprise-class storage management as software-defined storage (SDS). It is deployed and managed through the ONTAP Select Deploy administration utility, a separate Linux virtual machine.

### Repository structure

This repository is a flat structure — all AsciiDoc content files reside in the root directory. Files follow naming conventions that identify their type and topic area:

- `media/` – Images and diagrams referenced by AsciiDoc files
- `redirect/` – Redirect configuration for URL mapping
- `project.yml` – Site settings, version list, and sidebar navigation
- `_index.yml` – Landing page configuration

### Product-specific context

**Architecture and components:**
- *ONTAP Select node*: An ONTAP Select cluster is composed of one, two, four, six, eight, ten, or twelve nodes, each running as a separate virtual machine with a specialized version of ONTAP 9. All nodes in a cluster must run on the same hypervisor type and version.
- *ONTAP Select Deploy*: The administration utility used to deploy and manage ONTAP Select clusters. It runs as a dedicated Linux virtual machine and is the required tool for all production deployments. Deploy exposes three interfaces: a web UI, a CLI management shell (SSH), and a REST API. The online API documentation page is available at `http://<deploy_ip>/api/ui`.
- *License Manager (LM)*: A software component embedded within the Deploy utility that manages Capacity Pools licensing. Each Deploy instance is identified by a *License Lock ID (LLID)*, a numeric string required when generating Capacity Pool license files.
- *Mediator service*: A component of the Deploy utility that monitors two-node clusters and assists in resolving split-brain scenarios. The Deploy VM must remain running for two-node cluster HA to function. Communication uses iSCSI (port 3260) over IPv4.

**Storage architecture:**
- *Direct-attached storage (DAS)*: Local storage managed through a single RAID controller per host. Storage types include HDD (SAS, NL-SAS, SATA), SSD, and NVMe drives. All physical drives in an ONTAP Select managed storage configuration must be homogeneous (same type and speed).
- *Hardware RAID*: Traditional RAID functionality provided by a local hardware RAID controller. Physical disks are grouped into RAID groups, presented to the hypervisor as LUNs, and then configured as storage pools (datastores in ESXi).
- *Software RAID*: ONTAP Select provides the RAID functionality internally, removing the need for a hardware RAID controller. Only one ONTAP Select node can be deployed per host when using software RAID.
- *vNAS (Virtual NAS)*: An ONTAP Select solution for using external storage datastores. Supported configurations include VMware vSAN and generic external storage arrays (connected via FC, FCoE, iSCSI, or NFS). NFS external storage is not supported on KVM. vNAS enables VMware vMotion, HA, and DRS for ONTAP Select VMs.
- *Storage pool*: A logical data container abstracting underlying physical storage; hypervisor-independent. On ESXi, equivalent to a VMware *datastore*.

**Networking architecture:**
- *Internal network*: Present only in multi-node clusters. Carries intra-cluster traffic: cluster heartbeat, HA Interconnect (HA-IC), and RAID SyncMirror (RSM). Uses a single layer-2 VLAN, IPv4 only (no DHCP), with MTU configurable in the 7500–9000 byte range (default 9000).
- *External network*: Present in all cluster configurations. Carries data traffic (NFS, SMB/CIFS, iSCSI, NVMe over TCP), management, and optionally intercluster traffic. Supports IPv4 and IPv6.
- *Virtual switch*: On ESXi, the vSwitch (standard or distributed). On KVM, *Open vSwitch (OVS)*. Each ONTAP Select host must have a vSwitch configured before deployment.
- *VMXNET3*: The default network driver for ONTAP Select nodes on VMware ESXi.

**High availability:**
- *HA pair*: The basic HA unit; two nodes synchronously mirror data aggregates using RAID SyncMirror. HA cannot be used independently from synchronous replication in ONTAP Select.
- *Two-node cluster*: One HA pair requiring an external mediator (the Deploy VM). Node management IP must be IPv4; iSCSI is used for mediator communication.
- *Multi-node cluster*: Four, six, eight, ten, or twelve nodes (two, three, four, five, or six HA pairs). Quorum is maintained by surviving nodes without a mediator.
- *MetroCluster SDS*: A two-node stretched HA configuration where nodes are physically separated by more than 300m (in different rooms, buildings, or data centers). Requires a Premium or Premium XL license and a maximum 10ms round-trip latency between nodes (including shared storage latency).

**Licensing:**
- *Evaluation license*: Automatically generated and applied; allows evaluation of ONTAP Select without purchase.
- *Capacity Tiers*: A per-node licensing model where storage capacity is locked to the node and is perpetual (no renewal required). Separate license required for each node.
- *Capacity Pools*: A shared-pool licensing model where a license is locked to a License Manager (Deploy) instance and must be renewed. Capacity Pools are shared by ONTAP Select nodes, typically requiring fewer licenses than Capacity Tiers.
- *Platform license offerings*: Three tiers — **Standard**, **Premium**, and **Premium XL** — that determine the VM instance size (small, medium, or large) and supported storage types. Standard supports small instances with HDDs only (no SSD or NVMe with hardware RAID/software RAID). Premium adds medium instances, SSD support, and MetroCluster SDS. Premium XL adds large instances and NVMe support. KVM does not support the large instance type.
- *Instance sizes*: Small (4 vCPUs/16 GB RAM), Medium (8 vCPUs/64 GB RAM), Large (16 vCPUs/128 GB RAM).
- *NetApp License File (NLF)*: The license file format used for production licenses.

**Key concepts:**
- *Hypervisor host*: The physical server running ESXi or KVM that hosts one or more ONTAP Select VMs.
- *ONTAP Select node*: An ONTAP Select VM that is deployed and active on a hypervisor host.
- *Cluster refresh*: A feature to re-synchronize the Deploy configuration database with changes made to clusters or VMs outside of Deploy (for example, via ONTAP CLI or vSphere).
- *ONTAP Select image install*: The ability to add earlier ONTAP Select node images to a Deploy instance for deploying older releases. Only earlier versions than the Deploy version can be added.
- *Credential store*: A secure database within Deploy that holds hypervisor host and vCenter credentials, used for authentication during cluster creation and management.
- *Single Instance Data Logging (SIDL)*: A write optimization feature enabled by default in vNAS configurations.
- *Cluster MTU*: The MTU size for the internal cluster network, configurable between 7500–9000 bytes.

**Naming conventions and terminology:**
- *ONTAP Select* refers to the complete product (node software + Deploy utility)
- *ONTAP Select Deploy* or *Deploy* refers specifically to the administration utility VM
- *Deploy utility*, *Deploy administration utility*, and *Deploy VM* are used interchangeably in documentation
- *DAS* = direct-attached storage; *vNAS* = virtual NAS (external storage solution for ONTAP Select)
- *ROBO* = remote office/branch office (a key use case)
- *LLID* = License Lock ID (uniquely identifies a Deploy/License Manager instance)
- *RSM* = RAID SyncMirror (synchronous replication within an HA pair)
- *HA-IC* = High Availability Interconnect (internal HA heartbeat network)
- *OVS* = Open vSwitch (virtual switch used on KVM hosts)
- *MetroCluster SDS* is a software-only feature distinct from hardware-based MetroCluster; it is a stretched two-node cluster configuration

### Typical user workflows

**Initial deployment:**
Prepare hypervisor hosts (storage, networking, RAID) → Install ONTAP Select Deploy VM → Register hypervisor hosts in Deploy → Acquire and apply licenses → Deploy ONTAP Select cluster using Deploy web UI, CLI, or REST API → Verify initial cluster state

**Licensing a production cluster:**
Choose licensing model (Capacity Tiers or Capacity Pools) → Choose platform license offering (Standard/Premium/Premium XL) → Size required capacity → Purchase license from NetApp → Download license file (NLF) → Apply license using Deploy utility → Verify license in cluster

**Cluster administration:**
Sign in to Deploy web UI or CLI → Perform cluster operations (resize nodes, expand/contract cluster, replace drives) → Use ONTAP CLI or System Manager for ONTAP-level configuration → Run cluster refresh if changes were made outside Deploy

**Automating with the REST API:**
Access Deploy API at `http://<deploy_ip>/api/ui` → Review available API calls → Use Python scripts or curl to automate cluster creation, node licensing, cluster deletion, and node resizing

**Upgrading Deploy:**
Download new Deploy OVA → Deploy new Deploy VM → Migrate existing Deploy instance data to new VM → Verify cluster inventory → Continue managing clusters with upgraded Deploy
