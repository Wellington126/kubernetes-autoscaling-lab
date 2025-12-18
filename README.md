# ☸️ Kubernetes Lab: HPA & Stress Testing

Este projeto implementa um cluster Kubernetes local utilizando **Vagrant** e **VirtualBox**, configurado para demonstrar o **Horizontal Pod Autoscaling (HPA)** na prática.

O laboratório simula um cenário real onde uma aplicação sofre picos de carga de memória e o cluster reage provisionando novas réplicas automaticamente.

## 🏗 Arquitetura

* **Orquestrador:** Kubernetes v1.29 (Kubeadm)
* **OS:** Ubuntu Server 20.04 LTS
* **Nós:**
  * `k8s-master`: Control Plane (1.5GB RAM)
  * `k8s-worker-1`: Worker Node (1GB RAM)
  * `k8s-worker-2`: Worker Node (1GB RAM)

## 📋 Pré-requisitos

* Mínimo de **8GB de RAM** disponível no host.
* [VirtualBox](https://www.virtualbox.org/) e [Vagrant](https://www.vagrantup.com/) instalados.

---

## 🚀 Como Executar o Projeto

### 1. Clonar e Subir as VMs
```bash
git clone https://github.com/Wellington126/kubernetes-autoscaling-lab.git
cd lab-kubernetes
vagrant up
```

### 2. Instalação do Kubernetes (Roteiro Rápido)
Como as VMs sobem "limpas", execute os comandos abaixo para configurar o cluster.

**A. Em todos os nós (Master e Workers):**
Acesse via `vagrant ssh` e execute como root:
```bash
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
cat <<EOT | tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOT
modprobe overlay && modprobe br_netfilter
cat <<EOT | tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOT
sysctl --system

apt-get update && apt-get install -y containerd apt-transport-https ca-certificates curl gpg
mkdir -p /etc/containerd && containerd config default | tee /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
systemctl restart containerd

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list
apt-get update && apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl

# Fix de IP para Vagrant
IP_ADDR=$(ip -4 addr show eth1 | grep -oP '(?<=inet\s)\d+(\.\d+){3}')
echo "KUBELET_EXTRA_ARGS='--node-ip=${IP_ADDR}'" > /etc/default/kubelet
systemctl daemon-reload && systemctl restart kubelet
```

**B. Apenas no Master:**
```bash
kubeadm init --apiserver-advertise-address=192.168.56.10 --pod-network-cidr=192.168.0.0/16
# (Rode o comando kubeadm join nos workers depois disso)

# Configurar Kubectl e Rede
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml
```

---

### 3. Configurando o Teste de Autoscaling (No Master)

```bash
# Metrics Server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl patch deployment metrics-server -n kube-system --type='json' -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'

# Aplicação e HPA
kubectl apply -f stress.yaml
kubectl apply -f hpa.yaml
```

---

## 🧪 Executando o Teste de Estresse

1. **Monitore:** `watch kubectl get hpa`
2. **Gere carga:**
   ```bash
   kubectl exec -it $(kubectl get pod -l run=php-apache -o jsonpath="{.items[0].metadata.name}") -- stress-ng --vm 1 --vm-bytes 200M --timeout 600s
   ```
