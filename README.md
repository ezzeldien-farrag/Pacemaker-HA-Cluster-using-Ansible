# Pacemaker HA Cluster with iSCSI Storage

This Ansible project automates the setup of a high-availability Pacemaker cluster on CentOS with shared iSCSI storage, Apache web server, and NFS exports.

## Project Overview

This project creates:
- **3-node Pacemaker cluster** (`target1`, `target2`, `target3`)
- **iSCSI Target server** (`ansible-server`) providing shared storage
- **Floating IP** (192.168.33.1) for accessing the cluster
- **Apache web server** serving from shared storage
- **NFS server** exporting shared directories

The cluster ensures that services (Apache, NFS) run on a single active node while remaining nodes are standby. If the active node fails, Pacemaker automatically fails over to another node.

## Architecture

```
┌─────────────────┐
│ Ansible-Server  │ (iSCSI Target)
│ 192.168.33.20   │
│  - iSCSI LUN    │
│  - Shared disk  │
└────────┬────────┘
         │
         │ iSCSI
         ▼
┌──────────────────────────────────────────────────┐
│             3-Node Cluster                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐         │
│  │ target1  │  │ target2  │  │ target3 │         │
│  │          │  │          │  │          │         │
│  │ Pacemaker│─▶│ Pacemaker│─▶│ Pacemaker│         │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘         │
└───────┼─────────────┼─────────────┼───────────┘
        │             │               │
        └─────────────┴───────────────┘
                    │
              VIP: 192.168.33.1
                    │
              (One active node)
                    │
        ┌───────────┼───────────────┐
        ▼           ▼               ▼
     Apache    NFS Export   Shared Filesystem
              (from /srv/shared)
```

## Prerequisites

- **Host System**: Windows, Linux, or macOS with Vagrant and VirtualBox
- **Software**: 
  - Vagrant
  - VirtualBox (or other Vagrant provider)
  - Ansible 2.9+ installed locally
- **Network**: 192.168.33.0/24 subnet (Vagrant default)
- **Resources**: 
  - ~15GB disk space for VMs
  - 4GB+ RAM recommended
  - 4 CPUs recommended

## Inventory Configuration

The project uses an Ansible inventory (`inventory.ini`) with four hosts:

- `ansible-server`: iSCSI target server
- `target1`, `target2`, `target3`: Cluster nodes

All hosts use the `vagrant` user (default for Vagrant boxes).

## Configuration Variables

Edit `group_vars/all.yml` to customize:

- **VIP Address**: `vip_address` (default: 192.168.33.1)
- **iSCSI Portal IP**: `iscsi_portal_ip` (default: 192.168.33.20)
- **LUN Size**: `iscsi_lun_size_gb` (default: 5GB)
- **Shared Mount**: `shared_mount` (default: /srv/shared)
- **Cluster Password**: `hacluster_password` (default: clusterpass)

## Getting Started

### 1. Clone the Repository

```bash
git clone <repository-url>
cd pacemaker
```

### 2. Provision VMs (if using Vagrant)

```bash
vagrant up
```

This will provision 4 VMs:
- `ansible-server` - iSCSI target (IP: 192.168.33.20)
- `target1` - Cluster node 1 (IP: 192.168.33.10)
- `target2` - Cluster node 2 (IP: 192.168.33.11)
- `target3` - Cluster node 3 (IP: 192.168.33.12)

### 3. Run Ansible Playbooks

Execute the playbooks in order:

#### Step 1: Configure iSCSI Target
```bash
ansible-playbook -i inventory.ini 01-storage-target.yml
```

This playbook:
- Installs iSCSI target package (targetcli)
- Creates a 5GB file-backed LUN
- Configures the iSCSI target with ACLs for all cluster nodes

#### Step 2: Install Cluster Prerequisites
```bash
ansible-playbook -i inventory.ini 02-cluster-prereqs.yml
```

This playbook:
- Installs Pacemaker, Corosync, and pcs
- Configures iSCSI initiators on cluster nodes
- Sets SELinux to permissive mode
- Disables firewalld (for lab environment)
- Sets up `hacluster` user with password authentication

#### Step 3: Initialize Filesystem
```bash
ansible-playbook -i inventory.ini 03-fs-initial-mkfs.yml
```

This playbook:
- Connects all nodes to the iSCSI target
- Creates an ext4 filesystem on the shared LUN
- Creates service directories on the shared mount

#### Step 4: Configure Pacemaker Resources
```bash
ansible-playbook -i inventory.ini 04-pacemaker-resources.yml
```

This playbook:
- Creates and starts the Pacemaker cluster
- Configures cluster properties (STONITH disabled for lab)
- Creates resources:
  - VIP (192.168.33.1)
  - Shared filesystem mount
  - Apache web server
  - NFS server and export
- Sets up resource constraints (ordering and colocation)

## Verify the Setup

### Check Cluster Status

SSH to any cluster node and run:

```bash
pcs status
```

Expected output:
- Cluster name: **ZOZ**
- All nodes: **Online**
- One node should show resources running

### Check Resource Status

```bash
pcs resource status
```

You should see:
- `vip` - Floating IP (on one node)
- `fs_data` - Filesystem mount
- `apache` - Apache web server
- `nfsserver` - NFS server
- `export` - NFS export

### Test Failover

1. SSH to target1 and check which node has the VIP:
   ```bash
   ip addr show enp0s8 | grep 192.168.33.1
   ```

2. Simulate node failure:
   ```bash
   pcs node standby <active-node-name>
   ```

3. Watch the resources migrate:
   ```bash
   pcs status
   ```

4. Unstandby the node:
   ```bash
   pcs node unstandby <node-name>
   ```

## Accessing Services

### Apache Web Server

Visit: `http://192.168.33.1`

The document root is `/srv/shared/www` on all nodes and serves from the shared filesystem.

### NFS Export

Mount the export on a client machine:

```bash
mount -t nfs 192.168.33.1:/srv/shared/export /mnt
```

The NFS export is available at `/srv/shared/export` on the active node.

## Project Structure

```
.
├── 01-storage-target.yml      # Configure iSCSI target
├── 02-cluster-prereqs.yml     # Install cluster packages
├── 03-fs-initial-mkfs.yml     # Format shared filesystem
├── 04-pacemaker-resources.yml # Configure cluster resources
├── inventory.ini               # Ansible inventory
├── group_vars/
│   └── all.yml                # Global variables
└── README.md                  # This file
```

## Troubleshooting

### Cluster won't start
- Check pcsd service: `systemctl status pcsd`
- Verify network connectivity between nodes
- Check `pcs status` output for errors

### Resources show as "Blocked"
- Check SELinux: should be permissive
- Check firewalld: should be stopped (lab only)
- Review logs: `journalctl -xe`

### iSCSI connection issues
- Verify iSCSI target is running: `systemctl status target`
- Check initiator IQN: `cat /etc/iscsi/initiatorname.iscsi`
- Test connection: `iscsiadm -m session`

### Permissions issues
- Ensure hacluster password is set correctly
- Verify ACL entries in targetcli configuration

## Important Notes

⚠️ **Security Warning**: This configuration is for **LAB/TESTING ONLY**. It includes:
- SELinux set to permissive
- Firewalld disabled
- STONITH disabled
- Weak passwords

**DO NOT** use these settings in production!

## Production Considerations

For production deployment:
1. Enable STONITH fencing devices
2. Enable SELinux in enforcing mode
3. Configure firewalld with proper rules
4. Use strong passwords for hacluster
5. Implement proper backup strategy
6. Configure monitoring and alerting
7. Review and harden all security settings

## Additional Resources

- [Pacemaker Documentation](https://clusterlabs.org/pacemaker/doc/)
- [Corosync Documentation](https://corosync.github.io/corosync/)
- [iSCSI Target (targetcli) Documentation](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/managing_storage_devices/configuring-an-iscsi-target_managing-storage-devices)
- [PCS Command Line Interface](https://clusterlabs.org/pcs-manual.html)

## License

This project is provided as-is for educational and testing purposes.

