# Cortensor Multi-Node Setup Guide with Ansible

A comprehensive guide for deploying multiple Cortensor nodes on a single server using Ansible, based on real-world deployment experience and troubleshooting.

## Prerequisites

### System Requirements
- **OS**: Ubuntu 22.04 LTS
- **CPU**: Minimum 8 cores (12+ recommended for 4 nodes)
- **RAM**: Minimum 16GB (32GB recommended for 4 nodes)
- **Disk**: 200GB+ SSD
- **Network**: Stable internet connection

### Required Accounts
- 4 wallet addresses with staked COR tokens
  - **Note**: Staking requirements may change. Please check the latest announcements on [Cortensor Discord](https://discord.gg/cortensor) for current staking amount and period requirements
  - As of last update: 5000 COR tokens with 84 days staking period
- Private keys for each wallet

## Installation Steps

### 1. System Preparation

```bash
# Update system
apt update && apt upgrade -y

# Install prerequisites
apt install -y net-tools htop git curl wget
```

### 2. Install Ansible

```bash
# Install required packages
apt install -y software-properties-common git ssh python3-pip

# Add Ansible repository
add-apt-repository --yes --update ppa:ansible/ansible
apt install -y ansible-core

# Verify installation
ansible --version
```

### 3. Create Cortensor User and Directories

```bash
# Create user
useradd -m -s /bin/bash cortensor

# Create necessary directories
mkdir -p /home/cortensor/.cortensor
mkdir -p /home/cortensor/.cortensor/bin
mkdir -p /home/cortensor/.cortensor/logs
mkdir -p /home/cortensor/.cortensor/llm-files

# Set ownership
chown -R cortensor:cortensor /home/cortensor/.cortensor
```

### 4. Clone Cortensor Ansible Repository

```bash
cd /root
git clone https://github.com/cortensor/community-projects.git
cd community-projects/infras/cortensor-ansible
```

### 5. Configure Ansible Files

#### A. Configure ansible.cfg

```bash
cat > ansible.cfg << 'EOF'
[defaults]
host_key_checking = False
retry_files_enabled = False
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts
fact_caching_timeout = 3600
interpreter_python = auto_silent
roles_path = ./roles
EOF
```

#### B. Configure hosts.yml

```bash
cat > inventories/hosts.yml << 'EOF'
all:
  children:
    cortensor_nodes:
      hosts:
        CORA:
          ansible_host: 127.0.0.1
          ansible_connection: local
          key_pair_index: 1
          gpu_enabled: 0
          llm_threads: "2"
          llm_memory_index_dynamic_loading: 0
        CORB:
          ansible_host: 127.0.0.1
          ansible_connection: local
          key_pair_index: 2
          gpu_enabled: 0
          llm_threads: "2"
          llm_memory_index_dynamic_loading: 0
        CORC:
          ansible_host: 127.0.0.1
          ansible_connection: local
          key_pair_index: 3
          gpu_enabled: 0
          llm_threads: "2"
          llm_memory_index_dynamic_loading: 0
        CORD:
          ansible_host: 127.0.0.1
          ansible_connection: local
          key_pair_index: 4
          gpu_enabled: 0
          llm_threads: "2"
          llm_memory_index_dynamic_loading: 0
EOF
```

#### C. Configure keys.yml

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

# Unused keys (required by template)
public_key_5: "0x0000000000000000000000000000000000000000"
private_key_5: "0x0000000000000000000000000000000000000000000000000000000000000000"
EOF
```

#### D. Configure main.yml

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
ansible_ssh_private_key_file: ~/.ssh/id_ed25519
rpc_url: https://sepolia-arb-rpc.agnc.my.id/
eth_rpc_url: https://ethereum-rpc.publicnode.com
contract_address_runtime: "0x2ACb5EE389B06250cC40593edbCc6eF3b9cEC8c7"
cortensor_branch: main
node_prefix: "COR"

# Monitoring
enable_log_shipping: false
log_shipping_url: https://your.loki.endpoint/loki/api/v1/push
metric_shipping_url: https://your.metric.endpoint/api/v1/push

# Watchdog version
watcher_version: v1.0.2
EOF
```

### 6. Install IPFS

```bash
# Download and install IPFS
cd /tmp
wget https://github.com/ipfs/kubo/releases/download/v0.33.0/kubo_v0.33.0_linux-amd64.tar.gz
tar -xvzf kubo_v0.33.0_linux-amd64.tar.gz
cd kubo
sudo bash install.sh
```

### 7. Deploy Nodes

```bash
cd ~/community-projects/infras/cortensor-ansible

# Deploy using manager script
./manager.sh
# Select: 1 (Deploy Cortensor nodes)
# Select: 1 (All hosts)
# Press Enter for main branch
# Type: y (to confirm)
```

### 8. Fix Binary and Scripts

```bash
# Copy binary to correct location
cp /usr/local/bin/cortensord /home/cortensor/.cortensor/bin/
chown cortensor:cortensor /home/cortensor/.cortensor/bin/cortensord

# Create start script
cat > /home/cortensor/.cortensor/bin/start.sh << 'EOF'
#!/bin/bash
INSTANCE_ID=${1:-1}
NODE_DIR="/home/cortensor/.cortensor/node-${INSTANCE_ID}"

cd "$NODE_DIR"
exec /home/cortensor/.cortensor/bin/cortensord .env minerv4 2>&1
EOF

chmod +x /home/cortensor/.cortensor/bin/start.sh
chown cortensor:cortensor /home/cortensor/.cortensor/bin/start.sh
```

### 9. Setup Node Directories and Environment

```bash
# Create node directories
for i in 1 2 3 4; do
    mkdir -p /home/cortensor/.cortensor/node-${i}
    cp /home/cortensor/.cortensor/.env-${i} /home/cortensor/.cortensor/node-${i}/.env
    chown -R cortensor:cortensor /home/cortensor/.cortensor/node-${i}
    
    # Add CPU thread limit
    echo "LLM_OPTION_CPU_THREADS=2" >> /home/cortensor/.cortensor/node-${i}/.env
    
    # Disable IPFS (to avoid conflicts)
    echo "ENABLE_IPFS_SERVER=0" >> /home/cortensor/.cortensor/node-${i}/.env
done
```

### 10. Configure IPFS for Each Node

```bash
# Setup separate IPFS directories with different ports
for i in 1 2 3 4; do
    # Add IPFS path to environment
    echo "export IPFS_PATH=/home/cortensor/.ipfs-${i}" >> /home/cortensor/.cortensor/node-${i}/.env
    
    # Initialize IPFS
    sudo -u cortensor mkdir -p /home/cortensor/.ipfs-${i}
    sudo -u cortensor IPFS_PATH=/home/cortensor/.ipfs-${i} ipfs init
    
    # Configure different ports for each IPFS instance
    if [ -f /home/cortensor/.ipfs-${i}/config ]; then
        # API port (5001, 5002, 5003, 5004)
        sed -i "s/\"API\": \"\/ip4\/127.0.0.1\/tcp\/5001\"/\"API\": \"\/ip4\/127.0.0.1\/tcp\/500${i}\"/" /home/cortensor/.ipfs-${i}/config
        
        # Gateway port (8081, 8082, 8083, 8084)
        sed -i "s/\"Gateway\": \"\/ip4\/127.0.0.1\/tcp\/8080\"/\"Gateway\": \"\/ip4\/127.0.0.1\/tcp\/808${i}\"/" /home/cortensor/.ipfs-${i}/config
        
        # Swarm port (4001, 4002, 4003, 4004)
        sed -i "s/\/tcp\/4001/\/tcp\/400${i}/g" /home/cortensor/.ipfs-${i}/config
    fi
done
```

### 11. Create Systemd Services

```bash
# Create service files for each node
for i in 1 2 3 4; do
cat > /etc/systemd/system/cortensor-${i}.service << EOF
[Unit]
Description=Cortensor Service Instance ${i}
After=network.target

[Service]
Environment=PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/home/cortensor/.cortensor/bin
LimitNOFILE=1000000
MemorySwapMax=0
TimeoutStartSec=infinity
Type=simple
User=cortensor
Group=cortensor
StateDirectory=cortensor
Environment=CORTENSOR_PATH="/home/cortensor/.cortensor"
WorkingDirectory=/home/cortensor/.cortensor/node-${i}
EnvironmentFile=/home/cortensor/.cortensor/node-${i}/.env
StandardOutput=append:/var/log/cortensord-${i}.log
StandardError=append:/var/log/cortensord-${i}.log
ExecStart=/home/cortensor/.cortensor/bin/start.sh ${i}
Restart=on-failure
RestartSec=10
KillSignal=SIGINT

[Install]
WantedBy=multi-user.target
EOF
done

# Reload systemd
systemctl daemon-reload
```

### 12. Start Services

```bash
# Start nodes one by one to avoid conflicts
for i in 1 2 3 4; do
    systemctl enable cortensor-${i}
    systemctl start cortensor-${i}
    sleep 5
done

# Check status
systemctl status cortensor-1 cortensor-2 cortensor-3 cortensor-4 --no-pager | grep -E "Active:"
```

## Management Commands

### Check Status
```bash
# All nodes
systemctl status cortensor-1 cortensor-2 cortensor-3 cortensor-4

# Quick status check
for i in 1 2 3 4; do echo "Node $i: $(systemctl is-active cortensor-${i})"; done
```

### View Logs
```bash
# Live logs
journalctl -u cortensor-1 -f

# Check errors
tail -50 /var/log/cortensord-1.log
```

### Restart Services
```bash
# Restart all
systemctl restart cortensor-1 cortensor-2 cortensor-3 cortensor-4

# Restart single node
systemctl restart cortensor-1
```

### Stop Services
```bash
systemctl stop cortensor-1 cortensor-2 cortensor-3 cortensor-4
```

## Updating Nodes

### Binary Update
```bash
# Backup current binary
cp /usr/local/bin/cortensord /usr/local/bin/cortensord.backup.$(date +%Y%m%d)

# Download new binary (replace URL)
wget [NEW_BINARY_URL] -O /tmp/cortensord
chmod +x /tmp/cortensord

# Stop services
systemctl stop cortensor-1 cortensor-2 cortensor-3 cortensor-4

# Replace binary
cp /tmp/cortensord /usr/local/bin/cortensord
cp /tmp/cortensord /home/cortensor/.cortensor/bin/cortensord
chown cortensor:cortensor /home/cortensor/.cortensor/bin/cortensord

# Start services
systemctl start cortensor-1 cortensor-2 cortensor-3 cortensor-4
```

## Troubleshooting

### IPFS Lock Issues
```bash
pkill -9 -f ipfs
rm -f /home/cortensor/.ipfs*/repo.lock
systemctl restart cortensor-1 cortensor-2 cortensor-3 cortensor-4
```

### Node Restart Loop
1. Check logs: `tail -50 /var/log/cortensord-1.log`
2. Verify RPC connectivity
3. Check private keys are correct
4. Ensure sufficient system resources

### Resource Optimization
- Reduce CPU threads if system is overloaded
- Consider running fewer nodes if RAM is limited
- Monitor with `htop` and adjust accordingly

## System Requirements Per Node

- **CPU**: 2-3 threads per node
- **RAM**: 4-8GB per node
- **Network**: Stable connection required
- **Disk**: 50GB per node recommended

## Security Notes

- Keep private keys secure
- Regularly backup configuration files
- Monitor system logs for anomalies
- Keep nodes updated with latest releases

## Support

- Discord: https://discord.gg/cortensor
- Documentation: https://docs.cortensor.network
- GitHub: https://github.com/cortensor

---

*This guide is based on practical deployment experience and includes solutions for common issues encountered during setup.*
