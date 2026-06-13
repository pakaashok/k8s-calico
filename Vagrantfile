VAGRANTFILE_API_VERSION = "2"

BOX_NAME = "generic/ubuntu2204"

# Network layout
NETWORK_BASE = '192.168.56'
START_OCTET = 24

# Calico version pinning (single source of truth)
CALICO_VERSION = "v3.27.3"

HOSTS = {
  'master.kubedev.com'  => "#{NETWORK_BASE}.#{START_OCTET}",
  'worker1.kubedev.com' => "#{NETWORK_BASE}.#{START_OCTET + 1}",
  'worker2.kubedev.com' => "#{NETWORK_BASE}.#{START_OCTET + 2}"
}

$hosts_setup = <<-SCRIPT
set -e
echo "Configuring /etc/hosts..."
grep -qxF '#{NETWORK_BASE}.#{START_OCTET} master.kubedev.com master' /etc/hosts || echo '#{NETWORK_BASE}.#{START_OCTET} master.kubedev.com master' >> /etc/hosts
grep -qxF '#{NETWORK_BASE}.#{START_OCTET + 1} worker1.kubedev.com worker1' /etc/hosts || echo '#{NETWORK_BASE}.#{START_OCTET + 1} worker1.kubedev.com worker1' >> /etc/hosts
grep -qxF '#{NETWORK_BASE}.#{START_OCTET + 2} worker2.kubedev.com worker2' /etc/hosts || echo '#{NETWORK_BASE}.#{START_OCTET + 2} worker2.kubedev.com worker2' >> /etc/hosts
echo "/etc/hosts configured successfully"
SCRIPT

# ──────────────────────────────────────────────────────────
# CLEANUP: Remove stale join.sh BEFORE master initializes
# Runs ONLY on master, FIRST, before $setup
# ──────────────────────────────────────────────────────────
$cleanup_stale_join = <<'SCRIPT'
set -eux
echo "=========================================="
echo "Cleaning up any stale join.sh from /vagrant"
echo "=========================================="
rm -f /vagrant/join.sh
rm -f /vagrant/.join.ready
echo "Stale files removed (if any existed)"
SCRIPT

# ──────────────────────────────────────────────────────────
# COMMON SETUP: Runs on ALL nodes
# ──────────────────────────────────────────────────────────
$setup = <<'SCRIPT'
set -eux

NODE_IP=$1

echo "=========================================="
echo "Setting up Kubernetes node with IP: ${NODE_IP}"
echo "=========================================="

# Disable swap
swapoff -a
sed -i '/swap/d' /etc/fstab
systemctl disable --now systemd-zram-setup@zram0.service 2>/dev/null || true

# Fix mirror issues - use main Ubuntu archive
echo "Updating apt sources to use main Ubuntu archive..."
sed -i 's|mirrors.edge.kernel.org|archive.ubuntu.com|g' /etc/apt/sources.list
sed -i 's|http://|https://|g' /etc/apt/sources.list

# Clean apt cache
apt-get clean
rm -rf /var/lib/apt/lists/*

# Base tools with retry logic
echo "Running apt-get update with retry logic..."
MAX_RETRIES=5
RETRY_COUNT=0
UPDATE_SUCCESS=false

while [ $RETRY_COUNT -lt $MAX_RETRIES ]; do
    RETRY_COUNT=$((RETRY_COUNT + 1))
    echo "apt-get update attempt $RETRY_COUNT of $MAX_RETRIES..."

    if apt-get update -y; then
        UPDATE_SUCCESS=true
        echo "apt-get update succeeded!"
        break
    else
        echo "apt-get update failed, waiting 30 seconds before retry..."
        sleep 30
    fi
done

if [ "$UPDATE_SUCCESS" = false ]; then
    echo "ERROR: apt-get update failed after $MAX_RETRIES attempts"
    exit 1
fi

apt-get install -y apt-transport-https ca-certificates curl wget gnupg lsb-release \
  net-tools iproute2 bash-completion conntrack socat ipset ipvsadm tcpdump traceroute jq

# Containerd
VERSION="1.7.24"
echo "Installing containerd v${VERSION}..."
wget -q https://github.com/containerd/containerd/releases/download/v${VERSION}/containerd-${VERSION}-linux-amd64.tar.gz
tar Cxzvf /usr/local containerd-${VERSION}-linux-amd64.tar.gz
mkdir -p /usr/local/lib/systemd/system
wget -q -P /usr/local/lib/systemd/system https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
systemctl daemon-reload && systemctl enable --now containerd

# Runc
RUNC="v1.2.2"
echo "Installing runc ${RUNC}..."
wget -q https://github.com/opencontainers/runc/releases/download/${RUNC}/runc.amd64
install -m 755 runc.amd64 /usr/local/sbin/runc

# CNI plugins
CNI="v1.6.0"
echo "Installing CNI plugins ${CNI}..."
mkdir -p /opt/cni/bin
wget -q https://github.com/containernetworking/plugins/releases/download/${CNI}/cni-plugins-linux-amd64-${CNI}.tgz
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-${CNI}.tgz

# Containerd config with SystemdCgroup and correct sandbox image
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sed -i 's|sandbox_image = "registry.k8s.io/pause:.*"|sandbox_image = "registry.k8s.io/pause:3.9"|' /etc/containerd/config.toml
systemctl restart containerd

# Kernel modules
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
modprobe overlay
modprobe br_netfilter

# Sysctl settings for Kubernetes networking
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
net.ipv4.conf.all.forwarding        = 1
EOF
sysctl --system

# Kubernetes packages
echo "Installing Kubernetes packages..."
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | gpg --dearmor -o /etc/apt/trusted.gpg.d/k8s.gpg
echo "deb [signed-by=/etc/apt/trusted.gpg.d/k8s.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" > /etc/apt/sources.list.d/kubernetes.list

# Retry apt-get update for kubernetes repo
for i in $(seq 1 5); do
    apt-get update -y && break
    echo "apt-get update for k8s repo failed, retrying... (attempt $i/5)"
    sleep 10
done

apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl

# Configure kubelet to use correct node IP (critical for multi-NIC VMs)
mkdir -p /etc/default
echo "KUBELET_EXTRA_ARGS=--node-ip=${NODE_IP}" > /etc/default/kubelet

systemctl enable kubelet

echo "Pulling Kubernetes images..."
kubeadm config images pull

# ── SSH setup for password auth (so Git Bash/WSL can SSH directly) ──
echo "Configuring SSH for password authentication..."
echo "vagrant:vagrant" | chpasswd
sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
sed -i 's/^#*PubkeyAuthentication.*/PubkeyAuthentication yes/' /etc/ssh/sshd_config
systemctl restart ssh

echo "=========================================="
echo "Base setup completed for ${NODE_IP}"
echo "=========================================="
SCRIPT

# ──────────────────────────────────────────────────────────
# MASTER POST-SETUP
# Idempotent: works on first run AND re-runs
# ──────────────────────────────────────────────────────────
$master_post = <<'SCRIPT'
set -eux
NODE_IP=$1
CALICO_VERSION=$2

# ── Initialize cluster only if not already done ──
if [ ! -f /etc/kubernetes/admin.conf ]; then
  echo "=========================================="
  echo "Initializing Kubernetes Master"
  echo "=========================================="

  kubeadm init \
    --apiserver-advertise-address=${NODE_IP} \
    --pod-network-cidr=192.168.0.0/16 \
    --cri-socket unix:///run/containerd/containerd.sock \
    --control-plane-endpoint=${NODE_IP}:6443

  # Setup kubeconfig for root
  mkdir -p /root/.kube
  cp /etc/kubernetes/admin.conf /root/.kube/config
  chown root:root /root/.kube/config

  # Setup kubeconfig for vagrant user
  mkdir -p /home/vagrant/.kube
  cp /etc/kubernetes/admin.conf /home/vagrant/.kube/config
  chown vagrant:vagrant /home/vagrant/.kube/config

  echo "=========================================="
  echo "Installing Calico CNI ${CALICO_VERSION}"
  echo "=========================================="

  CALICO_DOWNLOADED=false
  for i in $(seq 1 5); do
    echo "Downloading Calico manifest... (attempt $i/5)"
    if curl -sL https://raw.githubusercontent.com/projectcalico/calico/${CALICO_VERSION}/manifests/calico.yaml \
      -o /tmp/calico.yaml; then
      CALICO_DOWNLOADED=true
      echo "Calico manifest downloaded successfully!"
      break
    fi
    echo "Failed to download Calico manifest, retrying..."
    sleep 15
  done

  if [ "$CALICO_DOWNLOADED" = false ]; then
    echo "ERROR: Could not download Calico manifest after 5 attempts"
    exit 1
  fi

  kubectl apply -f /tmp/calico.yaml

  # ── Install calicoctl on master ──
  echo "Installing calicoctl..."
  curl -sL https://github.com/projectcalico/calico/releases/download/${CALICO_VERSION}/calicoctl-linux-amd64 \
    -o /usr/local/bin/calicoctl
  chmod +x /usr/local/bin/calicoctl

  # Add calicoctl env vars to vagrant user's bashrc
  if ! grep -q "DATASTORE_TYPE" /home/vagrant/.bashrc; then
    cat <<'EOF' >> /home/vagrant/.bashrc

# Calicoctl environment
export DATASTORE_TYPE=kubernetes
export KUBECONFIG=/home/vagrant/.kube/config
alias calicoctl='sudo -E calicoctl'
EOF
  fi

else
  echo "=========================================="
  echo "Cluster already initialized — skipping kubeadm init"
  echo "Will regenerate join token below"
  echo "=========================================="
fi

# Ensure KUBECONFIG is set for any kubectl/kubeadm commands below
export KUBECONFIG=/etc/kubernetes/admin.conf

echo "=========================================="
echo "Waiting for Control Plane Components"
echo "=========================================="

sleep 15

# ── Wait for kube-controller-manager (writes JWS signatures) ──
echo "Waiting for kube-controller-manager to be Running..."
CM_READY=false
for i in $(seq 1 60); do
  CM_STATUS=$(kubectl get pods -n kube-system \
    -l component=kube-controller-manager \
    --no-headers 2>/dev/null | awk '{print $3}' | head -1)
  if [ "$CM_STATUS" = "Running" ]; then
    echo "✅ kube-controller-manager is Running!"
    CM_READY=true
    break
  fi
  echo "Waiting for kube-controller-manager... attempt $i/60 (status: ${CM_STATUS:-pending})"
  sleep 5
done

if [ "$CM_READY" = false ]; then
  echo "ERROR: kube-controller-manager never became Running"
  exit 1
fi

# Wait for CoreDNS
echo "Waiting for CoreDNS pods to be created..."
COREDNS_READY=false
for i in $(seq 1 60); do
  if kubectl get pods -n kube-system -l k8s-app=kube-dns 2>/dev/null | grep -q "coredns"; then
    echo "CoreDNS pods found, waiting for ready state..."
    if kubectl wait --for=condition=ready pod -l k8s-app=kube-dns \
      -n kube-system --timeout=300s 2>/dev/null; then
      COREDNS_READY=true
      echo "CoreDNS is ready!"
      break
    fi
  fi
  echo "Waiting for CoreDNS... attempt $i/60"
  sleep 5
done

if [ "$COREDNS_READY" = false ]; then
  echo "WARNING: CoreDNS did not become ready in time, but continuing..."
fi

# Wait for Calico node pods
echo "Waiting for Calico pods in kube-system namespace..."
CALICO_READY=false
for i in $(seq 1 60); do
  if kubectl get pods -n kube-system -l k8s-app=calico-node 2>/dev/null \
    | grep -q "calico-node"; then
    echo "Calico node pods found, waiting for ready state..."
    if kubectl wait --for=condition=ready pod -l k8s-app=calico-node \
      -n kube-system --timeout=300s 2>/dev/null; then
      CALICO_READY=true
      echo "Calico is ready!"
      break
    fi
  fi
  echo "Waiting for Calico... attempt $i/60"
  sleep 5
done

if [ "$CALICO_READY" = false ]; then
  echo "WARNING: Calico did not become ready in time, but continuing..."
fi

# Wait for calico-kube-controllers
echo "Waiting for Calico kube-controllers..."
for i in $(seq 1 30); do
  if kubectl get pods -n kube-system \
    -l k8s-app=calico-kube-controllers 2>/dev/null | grep -q "Running"; then
    echo "Calico kube-controllers is running!"
    break
  fi
  echo "Waiting for calico-kube-controllers... attempt $i/30"
  sleep 5
done

# Show final pod status
echo "=========================================="
echo "System pods status:"
echo "=========================================="
kubectl get pods -A

echo "=========================================="
echo "Node status:"
echo "=========================================="
kubectl get nodes -o wide

# ──────────────────────────────────────────────────────────
# JOIN TOKEN GENERATION — verified with JWS signature check
# ──────────────────────────────────────────────────────────
echo "=========================================="
echo "Cleaning stale tokens & generating verified join command"
echo "=========================================="

# 1. Remove any stale join.sh in synced folder (always)
rm -f /vagrant/join.sh
rm -f /vagrant/.join.ready

# 2. Delete ALL existing tokens to start fresh
echo "Deleting any existing tokens..."
for old_token in $(kubeadm token list 2>/dev/null | tail -n +2 | awk '{print $1}'); do
  echo "  Deleting old token: $old_token"
  kubeadm token delete "$old_token" 2>/dev/null || true
done

# 3. Generate fresh token + verify JWS signature is in cluster-info
TOKEN_VERIFIED=false
for attempt in $(seq 1 10); do
  echo "Token generation attempt $attempt/10..."

  JOIN_CMD=$(kubeadm token create --print-join-command 2>/dev/null || true)

  if [ -z "$JOIN_CMD" ]; then
    echo "Failed to generate join command, retrying in 10s..."
    sleep 10
    continue
  fi

  TOKEN_ID=$(echo "$JOIN_CMD" | grep -oP '[a-z0-9]{6}\.[a-z0-9]{16}' | cut -d. -f1)
  echo "Generated token ID: $TOKEN_ID"
  echo "Waiting for JWS signature to appear in cluster-info ConfigMap..."

  JWS_CONFIRMED=false
  for i in $(seq 1 24); do
    if kubectl get configmap cluster-info -n kube-public -o yaml 2>/dev/null \
      | grep -q "jws-kubeconfig-${TOKEN_ID}"; then
      JWS_CONFIRMED=true
      echo "✅ JWS signature confirmed for token ${TOKEN_ID}!"
      break
    fi
    echo "  Waiting for JWS signature... ($i/24)"
    sleep 5
  done

  if [ "$JWS_CONFIRMED" = true ]; then
    echo "$JOIN_CMD" > /vagrant/join.sh
    chmod +x /vagrant/join.sh
    echo "$TOKEN_ID" > /vagrant/.join.ready
    TOKEN_VERIFIED=true
    echo "✅ Verified join.sh written to /vagrant/join.sh"
    break
  else
    echo "JWS signature not found for token $TOKEN_ID, deleting & retrying..."
    kubeadm token delete "${TOKEN_ID}" 2>/dev/null || true
    sleep 10
  fi
done

if [ "$TOKEN_VERIFIED" = false ]; then
  echo "ERROR: Could not generate a verified join token after 10 attempts"
  exit 1
fi

echo "=========================================="
echo "Master initialization complete!"
echo "=========================================="
echo ""
echo "  📌 SSH: ssh vagrant@${NODE_IP}    (password: vagrant)"
echo "  📌 Kubectl: kubectl get nodes"
echo "  📌 Calicoctl: calicoctl node status"
echo ""
echo "=========================================="
SCRIPT

# ──────────────────────────────────────────────────────────
# WORKER JOIN
# Idempotent — handles re-provisioning cleanly
# ──────────────────────────────────────────────────────────
$worker_join = <<'SCRIPT'
set -eux

echo "=========================================="
echo "Joining worker to cluster"
echo "=========================================="

# Skip if already joined
if [ -f /etc/kubernetes/kubelet.conf ]; then
  echo "Worker already joined to a cluster — checking connectivity..."
  if systemctl is-active --quiet kubelet; then
    echo "✅ Kubelet is running and node is already joined. Skipping join."
    exit 0
  else
    echo "Kubelet not running — will reset and re-join"
    kubeadm reset -f --cri-socket unix:///run/containerd/containerd.sock 2>/dev/null || true
    rm -rf /etc/cni/net.d/* 2>/dev/null || true
    systemctl restart containerd
    sleep 5
  fi
fi

# Wait for verified join.sh
echo "Waiting for verified join.sh from master..."
JOIN_FOUND=false
for i in $(seq 1 120); do
  if [ -f /vagrant/join.sh ] && [ -f /vagrant/.join.ready ]; then
    JOIN_FOUND=true
    TOKEN_ID=$(cat /vagrant/.join.ready)
    echo "Found verified join.sh (token: $TOKEN_ID)"
    break
  fi
  echo "Waiting for verified join.sh... attempt $i/120"
  sleep 5
done

if [ "$JOIN_FOUND" = false ]; then
  echo "ERROR: verified join.sh not found after 10 minutes!"
  exit 1
fi

# Join the cluster
echo "Joining cluster..."
bash /vagrant/join.sh --cri-socket unix:///run/containerd/containerd.sock

echo "=========================================="
echo "Successfully joined the cluster!"
echo "=========================================="
SCRIPT


# ══════════════════════════════════════════════════════════
# VAGRANT CONFIGURATION
# ══════════════════════════════════════════════════════════
Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = BOX_NAME
  config.vm.synced_folder ".", "/vagrant"

  # ============================================
  # VMWARE DESKTOP PROVIDER (Global Settings)
  # ============================================
  config.vm.provider "vmware_desktop" do |v|
    v.gui = false
    v.linked_clone = true
    v.vmx["virtualHW.version"] = "19"
    v.vmx["mks.enable3d"] = "FALSE"
    v.vmx["sound.present"] = "FALSE"
    v.vmx["floppy0.present"] = "FALSE"
    v.vmx["scsi0:0.virtualSSD"] = "1"
    v.vmx["numvcpus"] = "2"
  end

  # ================== MASTER NODE ==================
  config.vm.define "master", primary: true do |m|
    m.vm.hostname = "master"
    m.vm.network :private_network, ip: "#{NETWORK_BASE}.#{START_OCTET}"

    # Port forwarding with auto_correct — Vagrant picks free port if collision
    m.vm.network "forwarded_port",
      guest: 31000, host: 31100, auto_correct: true, id: "kibana"
    m.vm.network "forwarded_port",
      guest: 30080, host: 30180, auto_correct: true, id: "http-app"
    m.vm.network "forwarded_port",
      guest: 30443, host: 31443, auto_correct: true, id: "https-app"

    m.vm.provider "vmware_desktop" do |v|
      v.vmx["displayName"] = "k8s-master"
      v.vmx["memsize"] = "4096"
      v.vmx["numvcpus"] = "2"
    end

    m.vm.provision "shell", inline: $hosts_setup
    m.vm.provision "shell", inline: $cleanup_stale_join
    m.vm.provision "shell", inline: $setup, args: ["#{NETWORK_BASE}.#{START_OCTET}"]
    m.vm.provision "shell", inline: $master_post,
      args: ["#{NETWORK_BASE}.#{START_OCTET}", CALICO_VERSION]
  end

  # ================== WORKER 1 ==================
  config.vm.define "worker1" do |w|
    w.vm.hostname = "worker1"
    w.vm.network :private_network, ip: "#{NETWORK_BASE}.#{START_OCTET + 1}"

    w.vm.provider "vmware_desktop" do |v|
      v.vmx["displayName"] = "k8s-worker1"
      v.vmx["memsize"] = "4096"
      v.vmx["numvcpus"] = "2"
    end

    w.vm.provision "shell", inline: $hosts_setup
    w.vm.provision "shell", inline: $setup, args: ["#{NETWORK_BASE}.#{START_OCTET + 1}"]
    w.vm.provision "shell", inline: $worker_join
  end

  # ================== WORKER 2 ==================
  config.vm.define "worker2" do |w|
    w.vm.hostname = "worker2"
    w.vm.network :private_network, ip: "#{NETWORK_BASE}.#{START_OCTET + 2}"

    w.vm.provider "vmware_desktop" do |v|
      v.vmx["displayName"] = "k8s-worker2"
      v.vmx["memsize"] = "4096"
      v.vmx["numvcpus"] = "2"
    end

    w.vm.provision "shell", inline: $hosts_setup
    w.vm.provision "shell", inline: $setup, args: ["#{NETWORK_BASE}.#{START_OCTET + 2}"]
    w.vm.provision "shell", inline: $worker_join
  end

end