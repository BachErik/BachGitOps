# Installation

## 1. Set Hostname

```bash
sudo hostnamectl set-hostname main
```

## 2. Setup resolv conf

```bash
# Use Cloudflare DNS for reliable resolution (important for cert-manager DNS-01 challenges)
# Debian 13 uses systemd-resolved by default
sudo mkdir -p /etc/systemd/resolved.conf.d

sudo tee /etc/systemd/resolved.conf.d/cloudflare.conf <<EOF
[Resolve]
DNS=1.1.1.1 1.0.0.1 2606:4700:4700::1111 2606:4700:4700::1064
FallbackDNS=8.8.8.8 8.8.4.4 2001:4860:4860::8888 2001:4860:4860::8844
DNSOverTLS=opportunistic
EOF

sudo systemctl restart systemd-resolved

# Verify
resolvectl status
```

## 3. Update System and install dependencies

```bash
sudo apt update && sudo apt upgrade -y
sudo apt full-upgrade -y
sudo apt install -y \
  curl \
  wget \
  git \
  gnupg2 \
  ca-certificates \
  apt-transport-https \
  bash-completion \
  htop \
  jq \
  unzip \
  wireguard-tools \
  open-iscsi \
  nfs-common \
  cryptsetup \
  dmsetup \
  ufw

# Choose your architecture (amd64 or arm64).
ARCH="amd64"

# Download the release binary.
curl -LO "https://github.com/longhorn/cli/releases/download/v1.11.1/longhornctl-linux-${ARCH}"

wget https://github.com/derailed/k9s/releases/latest/download/k9s_Linux_${ARCH}.tar.gz
tar xvf k9s_Linux_${ARCH}.tar.gz
chmod +x k9s
mv k9s /bin/
rm LICENSE README.md k9s_Linux_${ARCH}.tar.gz

# Enable and start iSCSI daemon
sudo systemctl enable iscsid --now

# Verify iSCSI is running
sudo systemctl status iscsid

# Load Longhorn-specific kernel modules (persist)
sudo tee /etc/modules-load.d/longhorn.conf <<EOF
dm_crypt
iscsi_tcp
EOF

sudo modprobe dm_crypt iscsi_tcp

# Verify
lsmod | grep -E "dm_crypt|iscsi_tcp"

# Check filesystem supports extents (should show ext4 or xfs)
df -Th /var/lib/longhorn 2>/dev/null || df -Th /

# FluxCD, Longhorn, and Kubernetes watches need higher limits than defaults
sudo tee /etc/sysctl.d/99-k3s.conf <<EOF
fs.inotify.max_user_instances = 8192
fs.inotify.max_user_watches   = 524288
EOF

sudo sysctl --system

# Verify
sysctl fs.inotify.max_user_instances fs.inotify.max_user_watches



ufw allow 22/tcp
ufw enable

sudo reboot now
```

## 4. k3s Installation

Beefy server:

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server" sh -s - \
  --cluster-init \
  --disable traefik \
  --disable local-storage \
  --disable-cloud-controller \
  --disable-helm-controller \
  --flannel-backend wireguard-native \
  --tls-san k3s.bacherik.de \
  --tls-san main.bacherik.de \
  --kube-apiserver-arg default-not-ready-toleration-seconds=15 \
  --kube-apiserver-arg default-unreachable-toleration-seconds=15 \
  --node-label node-role.bach-industries/heavy=true \
  --node-label topology.kubernetes.io/zone=node-1 \
  --write-kubeconfig-mode 644
```

light-servers:

```bash
export K3S_TOKEN="..."
export K3S_URL="https://main.bacherik.de:6443"

# Node 2
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server" sh -s - \
  --server "${K3S_URL}" \
  --token "${K3S_TOKEN}" \
  --disable traefik \
  --disable local-storage \
  --disable-cloud-controller \
  --disable-helm-controller \
  --flannel-backend wireguard-native \
  --tls-san k3s.bacherik.de \
  --tls-san main.bacherik.de \
  --kube-apiserver-arg default-not-ready-toleration-seconds=15 \
  --kube-apiserver-arg default-unreachable-toleration-seconds=15 \
  --node-label node-role.bach-industries/light=true \
  --node-label topology.kubernetes.io/zone=node-1
```

## 5. Copy kubeconfig

```bash
scp main:/etc/rancher/k3s/k3s.yaml ~/.kube/config
sed -i 's/127.0.0.1/k3s.bacherik.de/g' ~/.kube/config
```

## 5. Apply sops

```bash
kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey
```

## 6. FluxCD

```bash
export GITHUB_TOKEN=...
flux bootstrap github \
  --token-auth \
  --owner=BachErik \
  --repository=BachGitOps \
  --branch=main \
  --path=clusters/prod \
  --personal
```

# UFW

## Servers

```bash
ufw allow 80/tcp #HTTP
ufw allow 443/tcp #HTTPS

ufw allow 6443/tcp #apiserver
ufw allow 2379:2380/tcp #HA with embedded etcd
ufw allow 8472/udp #Flannel VXLAN
ufw allow 10250/tcp #Kubelet metrics and API
ufw allow 51820/udp #Required only for Flannel Wireguard with IPv4
ufw allow 51821/udp #Required only for Flannel Wireguard with IPv6
ufw allow 5001/tcp #Required only for embedded distributed registry (Spegel)
ufw allow 6443/tcp #Required only for embedded distributed registry (Spegel)
ufw allow 9100/tcp # Node exporter (Prometheus)
ufw allow from 10.42.0.0/16 to any #pods
ufw allow from 10.43.0.0/16 to any #services
```

## Agents

```bash
ufw allow 6443/tcp #apiserver
ufw allow 8472/udp #Flannel VXLAN
ufw allow 10250/tcp #Kubelet metrics and API
ufw allow 51820/udp #Required only for Flannel Wireguard with IPv4
ufw allow 51821/udp #Required only for Flannel Wireguard with IPv6
ufw allow 9100/tcp # Node exporter (Prometheus)
ufw allow 5001/tcp #Required only for embedded distributed registry (Spegel)
ufw allow 6443/tcp #Required only for embedded distributed registry (Spegel)
```
