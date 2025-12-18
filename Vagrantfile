Vagrant.configure("2") do |config|
  # Box padrão: Ubuntu 20.04 LTS (Focal)
  config.vm.box = "bento/ubuntu-20.04"
  
  # Configuração para desativar swap automaticamente (Ajuda no K8s)
  config.vm.provision "shell", inline: "swapoff -a"

  # --- MÁQUINA 1: CLIENTE / MASTER ---
  config.vm.define "client-master" do |master|
    master.vm.hostname = "k8s-master"
    master.vm.network "private_network", ip: "192.168.56.10"
    master.vm.provider "virtualbox" do |v|
      v.name = "k8s-master"
      v.memory = 2048  # 2GB RAM
      v.cpus = 2
    end
  end

  # --- MÁQUINA 2: WORKER 1 ---
  config.vm.define "worker1" do |w1|
    w1.vm.hostname = "k8s-worker-1"
    w1.vm.network "private_network", ip: "192.168.56.11"
    w1.vm.provider "virtualbox" do |v|
      v.name = "k8s-worker-1"
      v.memory = 1536  # 1.5GB RAM
      v.cpus = 1
    end
  end

  # --- MÁQUINA 3: WORKER 2 ---
  config.vm.define "worker2" do |w2|
    w2.vm.hostname = "k8s-worker-2"
    w2.vm.network "private_network", ip: "192.168.56.12"
    w2.vm.provider "virtualbox" do |v|
      v.name = "k8s-worker-2"
      v.memory = 1536  # 1.5GB RAM
      v.cpus = 1
    end
  end
end
