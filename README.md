# Install-kubernetes
Instalación de un cluster  de Kubernetes de manera manual

Prerrequisitos de sistema operativo:
1- systemctl status firewalld.service
2- vim /etc/fstab comentar la linea siguiente:
![image](https://github.com/user-attachments/assets/e0285564-cd85-4d0f-8a06-05c259487099)

3- cambiar SELINUX a modo permisivo para que los contenedores puedan acceder a los archivos del sistema. 

sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

4- El siguiente comando configura el reenvio de paquetes entre servidores:

cat <<EOF | tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

5- Instalacion de containerd:

sudo dnf remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine \
                  podman \
                  runc
                  
- sudo dnf -y install dnf-plugins-core
- sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
- en la ruta /etc/containerd agregamos el archivo config.toml adjunto
- systemctl enable --now containerd

6- Mapear el repo de kubernetes:

cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

7- instalar kubelet, kubectl y kubeadm

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

8- iniciar cluster nodo principal

kubeadm init --control-plane-endpoint=10.140.3.191:6443 --pod-network-cidr "10.244.0.0/16

10.140.3.191 es la ip flotante que balancea los nodos ha-proxy

9- Unir los demas nodos master:

sudo kubeadm join 10.140.3.191:6443 --token pnyi2u.yu5lwwrdb5kfrkzv --discovery-token-ca-cert-hash sha256:07fc56078e3c5b5368fecd636baa4459b0d4a146faf9730283735ba7e85285c0  --control-plane --certificate-key c4f24d58b2557e8a24948d9c31206bb5e56e5bcc808fda361e6c39c37c337a03
El certificate-key lo obtenemos con el comando:

sudo kubeadm init phase upload-certs --upload-certs

10- Habilitar permisos para ejecucion de comandos kubectl. hacerlo en los 3 nodos master:

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

11- Instalación del complemento de red:

sudo curl https://raw.githubusercontent.com/projectcalico/calico/v3.29.2/manifests/calico.yaml -O
vim calico.yaml
buscar la sesion CALICO_IPV4POOL_CIDR e indicarl el CIDR utilizado para la instalación del cluster:
![image](https://github.com/user-attachments/assets/ec6af479-8f08-483a-baf7-1713a240abf8)

kubectl apply -f calico.yaml






