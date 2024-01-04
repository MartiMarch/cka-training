1. Instalar herramientas básicas

   # https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/
   sudo apt -y install vim
   sudo apt -y install apt-transport-https
   sudo apt -y install ca-certificates
   sudo apt -y install curl
   sudo apt -y install gpg

2. Configuración IP forwarding

   # https://kubernetes.io/docs/setup/production-environment/container-runtimes/#install-and-configure-prerequisites
   sudo sh -c 'echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf'
   sudo sh -c 'echo "net.bridge.bridge-nf-call-iptables=1" >> /etc/sysctl.conf'
   sudo sh -c 'echo "net.bridge.bridge-nf-call-ip6tables=1" >> /etc/sysctl.conf'
   sudo sysctl --system
   modprobe br_netfilter

3. Deshabilitando swap

   sudo sysctl vm.swappiness=0
   sudo sh -c 'echo "vm.swappiness=0" >> /etc/fstab'
   sudo sudo sed -i '/\/swap.img/d' /etc/fstab
   sudo swapoff -a

4. Instalación de containerd

   ###
   # Containerd
   # https://github.com/containerd/containerd/blob/main/docs/getting-started.md
   # https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd
   ##

   sudo systemctl enable containerd
   sudo systemctl install containerd
   sudo mkdir -p /etc/containerd
   sudo touch /etc/containerd/config.toml
   containerd config default
   sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
   sudo service containerd restart

5. Instalación kubectl

   ##
   # Kubectl
   # https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/
   ##
   curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
   echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
   sudo apt -y update
   sudo apt -y install kubectl