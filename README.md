# 📘 Instalación de un cluster  de Kubernetes K8S manera manual

## 📌Prerrequisitos de sistema operativo:
 * Desactivar firewall del servidor.
```
systemctl disable firewalld.service
```
  * Desactivar la swap
```
vim /etc/fstab
```
Comentamos la siguiente linea:

![image](https://github.com/user-attachments/assets/e0285564-cd85-4d0f-8a06-05c259487099)

  * Cambiar SELINUX a modo permisivo para que los contenedores puedan acceder a los archivos del sistema. 
```
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```
  * El siguiente comando configura el reenvio de paquetes entre servidores.
```
cat <<EOF | tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```
## 📌 Instalacion del runtime, en este caso intalaremos containerd:

  * Desinstalar todo rastro de docker o otro runtime en el servidor.
```
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
```
  * Instalar algunos plugines necesarios.
```                  
sudo dnf -y install dnf-plugins-core
```
  * Configurar el repositorio  de docker para realizar la descarga de los paquetes.
```
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
```
  * En la ruta /etc/containerd agregamos el archivo config.toml adjunto en este repo

  * Habilitamos el servicio para que inicie con el sistema opeartivo.
```
systemctl enable --now containerd
```
## 📌 Instalacion de kubernetes:

  * Configurar repositorio para descargar paquetes de Kubernetes
```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```
  * Instalar kubelet, kubectl y kubeadm
```
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```
  * Despues de ejecutar los pasos anteriores desactivar el proxy de los servidores (si tiene alguno activo) para porder arrancar el master.
    
  * Iniciar cluster nodo principal.
kubeadm init --control-plane-endpoint=10.140.3.191:6443 --pod-network-cidr "10.244.0.0/16
````
10.140.3.191 es la ip flotante que balancea los nodos ha-proxy

  * Unir los demas nodos master:
```
sudo kubeadm join 10.140.3.191:6443 --token pnyi2u.yu5lwwrdb5kfrkzv --discovery-token-ca-cert-hash sha256:07fc56078e3c5b5368fecd636baa4459b0d4a146faf9730283735ba7e85285c0  --control-plane --certificate-key c4f24d58b2557e8a24948d9c31206bb5e56e5bcc808fda361e6c39c37c337a03
```
El certificate-key lo obtenemos con el comando:
```
sudo kubeadm init phase upload-certs --upload-certs
```
 * Habilitar permisos para ejecucion de comandos kubectl. hacerlo en los 3 nodos master:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Nota: No dar permisos al usuario root para kubectl.

## 📌Instalación del complemento de red:

sudo curl https://raw.githubusercontent.com/projectcalico/calico/v3.29.2/manifests/calico.yaml -O
vim calico.yaml
buscar la sesion CALICO_IPV4POOL_CIDR e indicarl el CIDR utilizado para la instalación del cluster:


![image](https://github.com/user-attachments/assets/ec6af479-8f08-483a-baf7-1713a240abf8)

kubectl apply -f calico.yaml






