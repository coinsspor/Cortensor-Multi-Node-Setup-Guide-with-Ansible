# Cortensor Multi-Node Setup Guide

A simple, straightforward guide for running multiple Cortensor nodes on a single Ubuntu server.

## System Requirements

- **OS:** Ubuntu 22.04 LTS
- **CPU:** 12+ cores recommended
- **RAM:** 32GB minimum
- **Disk:** 200GB SSD
- **Network:** Stable internet connection

## Prerequisites

- 4 wallet addresses with staked COR tokens
- Private keys for each wallet
- Root access to the server

## Installation Steps

### Step 1: Clean Previous Installation (if exists)

```bash
# Stop and remove old services
systemctl stop cortensor* 2>/dev/null
systemctl disable cortensor* 2>/dev/null
rm -f /etc/systemd/system/cortensor*
systemctl daemon-reload

# Remove old files
rm -rf /home/cortensor/.cortensor
rm -rf /root/community-projects
rm -f /usr/local/bin/cortensord*
pkill -9 -f ipfs 2>/dev/null
rm -rf /home/cortensor/.ipfs*
```

### Step 2: Install Ansible

```bash
apt update
apt install -y software-properties-common
add-apt-repository --yes --update ppa:ansible/ansible
apt install -y ansible-core
ansible-galaxy collection install community.general
```

### Step 3: Clone Repository

```bash
cd /root
git clone https://github.com/cortensor/community-projects.git
cd community-projects/infras/cortensor-ansible
```

### Step 4: Configure hosts.yml

```bash
cat > inventories/hosts.yml << 'EOF'
all:
  children:
    cortensor_nodes:
      hosts:
        node1:
          ansible_host: 127.0.0.1
          ansible_connection: local
          key_pair_index: 1
          gpu_enabled: 0
          llm_threads: "2"
        node2:
          ansible_host: 127.0.0.1
          ansible_connection: local
          key_pair_index: 2
          gpu_enabled: 0
          llm_threads: "2"
        node3:
          ansible_host: 127.0.0.1
          ansible_connection: local
          key_pair_index: 3
          gpu_enabled: 0
          llm_threads: "2"
        node4:
          ansible_host: 127.0.0.1
          ansible_connection: local
          key_pair_index: 4
          gpu_enabled: 0
          llm_threads: "2"
EOF
```

### Step 5: Configure keys.yml

Replace with your actual wallet addresses and private keys:

```bash
cat > inventories/group_vars/all/keys.yml << 'EOF'
---
# Node 1
public_key_1: "0xYOUR_WALLET_ADDRESS_1"
private_key_1: "0xYOUR_PRIVATE_KEY_1"

# Node 2
public_key_2: "0xYOUR_WALLET_ADDRESS_2"
private_key_2: "0xYOUR_PRIVATE_KEY_2"

# Node 3
public_key_3: "0xYOUR_WALLET_ADDRESS_3"
private_key_3: "0xYOUR_PRIVATE_KEY_3"

# Node 4
public_key_4: "0xYOUR_WALLET_ADDRESS_4"
private_key_4: "0xYOUR_PRIVATE_KEY_4"

# Unused (required by template)
public_key_5: "0x0000000000000000000000000000000000000000"
private_key_5: "0x0000000000000000000000000000000000000000000000000000000000000000"
EOF
```

### Step 6: Configure main.yml

```bash
cat > inventories/group_vars/all/main.yml << 'EOF'
# Cortensor Variables
cortensor_user: cortensor
cortensor_home: "/home/{{ cortensor_user }}/.cortensor"
cortensor_bin: "{{ cortensor_home }}/bin"
cortensor_logs: "{{ cortensor_home }}/logs"

# IPFS Version
ipfs_version: "v0.33.0"
ipfs_package: "kubo_{{ ipfs_version }}_linux-amd64.tar.gz"
ipfs_url: "https://github.com/ipfs/kubo/releases/download/{{ ipfs_version }}/{{ ipfs_package }}"

# Installer repository
installer_repo: "https://github.com/cortensor/installer"
temp_dir: "/tmp/cortensor-install"

# Server configuration
ansible_user: root
ansible_ssh_private_key_file: ~/.ssh/id_rsa
rpc_url: https://sepolia-rollup.arbitrum.io/rpc
eth_rpc_url: https://ethereum-rpc.publicnode.com
contract_address_runtime: "0x8d67608D0F674F359DE0e420857739ECBDeb6B90"
llm_memory_index_dynamic_loading: 0
ipfs_primary: true
cortensor_branch: main
node_prefix: "COR"

# Monitoring (disabled)
enable_log_shipping: false
log_shipping_url: https://your.loki.endpoint/loki/api/v1/push
metric_shipping_url: https://your.metric.endpoint/api/v1/push

# Watchdog version
watcher_version: v1.0.2
EOF
```

### Step 7: Deploy Nodes

```bash
# Give execution permission to manager script
chmod +x manager.sh

# Run deployment
./manager.sh
```

When prompted:
1. Select `1` - Deploy Cortensor nodes
2. Select `1` - All hosts
3. Press Enter (for main branch)
4. Type `y` to confirm

### Step 8: Fix IPFS Conflicts (Important!)

After deployment, disable IPFS to prevent lock conflicts:

```bash
# Stop all nodes
systemctl stop cortensor-1 cortensor-2 cortensor-3 cortensor-4

# Disable IPFS for all nodes
for i in 1 2 3 4; do
    echo "ENABLE_IPFS_SERVER=0" >> /home/cortensor/.cortensor/.env-${i}
done

# Clean IPFS locks
pkill -9 -f ipfs
rm -f /home/cortensor/.ipfs/repo.lock

# Start nodes
systemctl start cortensor-1 cortensor-2 cortensor-3 cortensor-4
```

### Step 9: Verify Installation

```bash
# Check node status
for i in 1 2 3 4; do
    STATUS=$(systemctl is-active cortensor-${i})
    echo "Node $i: $STATUS"
done

# Check logs for errors
for i in 1 2 3 4; do
    echo "Node $i errors:"
    tail -5 /var/log/cortensord-${i}.log | grep -i "error" || echo "No errors"
done
```

## Management Commands

### Check Status
```bash
systemctl status cortensor-1 cortensor-2 cortensor-3 cortensor-4
```

### Restart Nodes
```bash
systemctl restart cortensor-1 cortensor-2 cortensor-3 cortensor-4
```

### Stop Nodes
```bash
systemctl stop cortensor-1 cortensor-2 cortensor-3 cortensor-4
```

### View Logs
```bash
# Live logs for node 1
journalctl -u cortensor-1 -f

# Check specific node log
tail -50 /var/log/cortensord-1.log
```

### Adjust CPU Threads

To reduce CPU usage, lower thread count:

```bash
# Stop nodes
systemctl stop cortensor-1 cortensor-2 cortensor-3 cortensor-4

# Change threads (example: set to 2)
for i in 1 2 3 4; do
    sed -i 's/LLM_OPTION_CPU_THREADS=.*/LLM_OPTION_CPU_THREADS=2/g' /home/cortensor/.cortensor/.env-${i}
done

# Restart nodes
systemctl start cortensor-1 cortensor-2 cortensor-3 cortensor-4
```

## Troubleshooting

### Nodes Keep Restarting

Check for IPFS lock issues:
```bash
# Check logs
grep "ipfs" /var/log/cortensord-1.log

# If IPFS lock errors exist, disable IPFS as shown in Step 8
```

### High CPU Usage

Reduce CPU threads:
```bash
for i in 1 2 3 4; do
    sed -i 's/LLM_OPTION_CPU_THREADS=.*/LLM_OPTION_CPU_THREADS=1/g' /home/cortensor/.cortensor/.env-${i}
done
systemctl restart cortensor-1 cortensor-2 cortensor-3 cortensor-4
```

### Version Mismatch Errors

Add version check bypass:
```bash
for i in 1 2 3 4; do
    echo "DISABLE_VERSION_CHECK=true" >> /home/cortensor/.cortensor/.env-${i}
done
```

## Important Notes

1. **IPFS Conflict**: When running multiple nodes on one server, IPFS must be disabled to prevent lock conflicts
2. **CPU Threads**: Adjust based on your system. Start with 2 threads per node
3. **RPC**: Uses public Arbitrum Sepolia RPC. For better performance, consider getting your own RPC endpoint
4. **Runtime Address**: Currently set for Dev7. Check Discord for updates

## Support

- Discord: https://discord.gg/cortensor
- Documentation: https://docs.cortensor.network
- GitHub: https://github.com/cortensor

---

*Last tested: September 2025 with Cortensor version 0.0.1-05a9c16*
