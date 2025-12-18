# ☸️ Kubernetes Lab: Autoscaling (HPA) & Stress Testing

![Kubernetes](https://img.shields.io/badge/Kubernetes-1.29-blue?logo=kubernetes)
![Vagrant](https://img.shields.io/badge/Vagrant-VirtualLab-1868F2?logo=vagrant)
![VirtualBox](https://img.shields.io/badge/VirtualBox-Hypervisor-183A61?logo=virtualbox)

> Laboratório prático para demonstrar **Horizontal Pod Autoscaling (HPA)** em Kubernetes sob **estresse de memória**, utilizando ambiente **On-Premise virtualizado**.

Este projeto implementa um cluster Kubernetes **On-Premise** virtualizado utilizando **Vagrant** e **VirtualBox**. O objetivo é demonstrar, na prática, o funcionamento do **Horizontal Pod Autoscaling (HPA)** sob condições de estresse de memória.

## 📑 Índice

* [Visão Geral](#-kubernetes-lab-autoscaling-hpa--stress-testing)
* [Arquitetura e Topologia](#-1-arquitetura-e-topologia)
* [Tecnologias Utilizadas](#stack-utilizada)
* [Instalação e Execução](#-2-guia-de-instalação-e-execução)
* [Preparação do Teste](#-3-preparação-do-teste-master)
* [Teste de Estresse](#-4-execução-do-teste-de-estresse)
* [Resultados Esperados](#-resultado-esperado)


---

## 🏗 1. Arquitetura e Topologia

O ambiente é composto por **3 Máquinas Virtuais** rodando **Ubuntu Server 20.04 LTS**:

| Hostname         | Função        | Specs (Lab)       | IP (Bridge)   |
| :--------------- | :------------ | :---------------- | :------------ |
| **k8s-master**   | Control Plane | 2 vCPU, 1.5GB RAM | 192.168.56.10 |
| **k8s-worker-1** | Worker Node   | 1 vCPU, 1.0GB RAM | 192.168.56.11 |
| **k8s-worker-2** | Worker Node   | 1 vCPU, 1.0GB RAM | 192.168.56.12 |

**Stack utilizada:**

* ☸️ Kubernetes v1.29 (kubeadm)
* 📦 Container Runtime: Containerd
* 📊 Monitoramento: Metrics Server + K9s
* 🌐 CNI: Calico

---

## 🚀 2. Guia de Instalação e Execução

### 2.1. Provisionamento do Ambiente

```bash
# Clone o repositório
git clone https://github.com/Wellington126/kubernetes-autoscaling-lab.git
cd lab-kubernetes

# Inicie as máquinas virtuais
vagrant up
```

---

### 2.2. Instalação do Kubernetes (Todos os Nós)

Acesse cada VM via `vagrant ssh` e execute como **root** (`sudo -i`).

```bash
# === 1. Configurações de Sistema ===
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

cat <<EOF | tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

modprobe overlay && modprobe br_netfilter

cat <<EOF | tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sysctl --system

# === 2. Instalar Containerd ===
apt-get update && apt-get install -y containerd apt-transport-https ca-certificates curl gpg
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
systemctl restart containerd

# === 3. Instalar Kubeadm, Kubelet e Kubectl (v1.29) ===
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' \
| tee /etc/apt/sources.list.d/kubernetes.list

apt-get update && apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl

# === 4. Correção de Rede (Vagrant) ===
IP_ADDR=$(ip -4 addr show eth1 | grep -oP '(?<=inet\s)\d+(\.\d+){3}')
echo "KUBELET_EXTRA_ARGS='--node-ip=${IP_ADDR}'" > /etc/default/kubelet

systemctl daemon-reload && systemctl restart kubelet
```

---

### 2.3. Inicialização do Cluster (Apenas no Master)

```bash
# Inicializar o Control Plane
kubeadm init \
  --apiserver-advertise-address=192.168.56.10 \
  --pod-network-cidr=192.168.0.0/16

# Configurar kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Instalar Calico
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml
```

➡️ Após isso, execute o comando `kubeadm join` nos **Workers**.

---

## ⚙️ 3. Preparação do Teste (Master)

### 3.1. Metrics Server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

kubectl patch deployment metrics-server -n kube-system \
  --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'
```

---

### 3.2. K9s (Monitoramento via Terminal)

```bash
curl -sS https://webinstall.dev/k9s | bash
source ~/.config/envman/PATH.env
```

---

### 3.3. Deploy da Aplicação e do HPA

```bash
kubectl apply -f stress.yaml
kubectl apply -f hpa.yaml
```

---

## 🧪 4. Execução do Teste de Estresse

O teste injeta carga de memória **acima do limite configurado no HPA (64MB)**, forçando o escalonamento automático.

### Passo 1: Monitorar

```bash
k9s
```

### Passo 2: Gerar Estresse

```bash
kubectl exec -it \
$(kubectl get pod -l run=php-apache -o jsonpath="{.items[0].metadata.name}") \
-- stress-ng --vm 1 --vm-bytes 200M --timeout 600s
```

📈 Durante o teste, observe:

* Aumento do consumo de memória
* Criação automática de novos Pods
* Balanceamento da carga entre os Workers

---

## 📊 Resultado Esperado

* O **HPA detecta o uso excessivo de memória**
* Novos Pods são criados automaticamente
* O cluster mantém estabilidade mesmo sob estresse

---
📌 **Laboratório ideal para estudos de:**

* Autoscaling
* Observabilidade
* Kubernetes On-Premise
* Testes de desempenho
