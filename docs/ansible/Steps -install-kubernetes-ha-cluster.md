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
  sudo modprobe br_netfilter

3. Deshabilitando swap

   sudo sysctl vm.swappiness=0
   sudo sed -i '/\/swap.img/d' /etc/fstab
   sudo swapoff -a

4. Instalación de containerd

  ###
  # Containerd
  # https://github.com/containerd/containerd/blob/main/docs/getting-started.md
  # https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd
  ##
  sudo apt -y install containerd
  sudo systemctl enable containerd
  sudo systemctl start containerd
  sudo mkdir -p /etc/containerd
  sudo touch /etc/containerd/config.toml
  sudo containerd config default > /etc/containerd/config.toml
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

6. Instalación del balanceador

  apt -y install haproxy
  vi /etc/haproxy/haproxy.cfg:
    frontend kube-apiserver
      bind *:<puerto balanceador>
      mode tcp
      option tcplog
      default_backend kube-apiserver

    backend kube-apiserver
      mode tcp
      option tcplog
      option tcp-check
      balance roundrobin
      default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
      server kube-apiserver-1 <ip controlador 1>:6443 check
      server kube-apiserver-2 <ip controlador 2>:6443 check
      server kube-apiserver-3 <ip controlador 3>:6443 check
  systemctl start haproxy
  systemctl enable haproxy
  apt -y install keepalived
  vi /etc/keepalived/keepalived.conf:
    global_defs {
      notification_email {}
      router_id LVS_DEVEL
      vrrp_skip_check_adv_addr
      vrrp_garp_interval 0
      vrrp_gna_interval 0
    }
    vrrp_script chk_haproxy {
      script "killall -0 haproxy"
      interval 2
      weight 2
    }
    vrrp_instance haproxy-vip {
      state BACKUP
      priority 100
      interface <interfaz de red>
      virtual_router_id 60
      advert_int 1
      authentication {
        auth_type PASS
        auth_pass 1111
      }
      unicast_src_ip <ip de la interfaz de red>
      unicast_peer {
        <Lista de IP de los otros nodos separadas por un salto de línea>
      }
      virtual_ipaddress {
        <IP virtual que se creará>/<subred de la ip>
      }
      track_script {
        chk_haproxy
      }
    }
  systemctl start keepalived
  systemctl enable keepalived

7. Preparación nodo controller

  sudo apt -y install kubelet
  sudo apt -y install kubeadm
  sudo systemctl daemon-reload
  sudo systemctl start kubelet

8. Inicialización del cluster (controller)

  sudo kubeadm init --pod-network-cidr=<subred kubernetes>/16 --control-plane-endpoint <ip virtual>:<puerto ip virtual> --upload-certs

9. Configuración de kubectl en consola administrador

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

10. Instalación plugin red

  kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
