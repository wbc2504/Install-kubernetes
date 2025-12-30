# <img width="50" height="30" alt="image" src="https://github.com/user-attachments/assets/e62c4f1b-b773-4502-9d49-d7adb3163929" /> Instalación de un cluster de Kubernetes k8s con kubeadm 

Paso a paso para la instalación de kubernetes con kubeadm. Se instalara un cluster de alta disponibilidad con 3 nodos master y 4 nodos worker, utilizando HAPROXY para balanceo de los master y a su vez keepalive para generar alta disponibilidad a estos ultimos.

 * Sistema operativo Rocky Linux 9.6 (Blue Onyx)
 * Kubernetes v1.32.5

## <img width="50" height="30" alt="image" src="https://github.com/user-attachments/assets/ec49154f-6d7d-4902-b304-8fb99a8c24dd" /> Instalacion de los HAPROXY

##### Se realiza la instalación del servicio de HAPROXY y keepalived en los dos servidores, tanto el activo como el pasivo.

```
yum install keepalived haproxy psmisc -y
```
En la ruta /etc/haproxy se crea el archivo de configuración haproxy.cfg tomando como base el que esta adjunto en el proyecto  y en la ruta /etc/keepalived tambien se crear el archivo de configuración keepalived.conf con base al adjunto en el proyecto. 

#### Se habilitan los servicios.

```
systemctl enable haproxy
systemctl enable keepalived
```

#### Se arranca el servicio de keepalived en ambos servidores.

```
systemctl start keepalived
```
Se comprueba que en el nodo activo (MASTER) se encuentre mapeada la ip virtual sobre la interfaz del servidor.

```
ip a
```
#### Se arranca el servicio de HAPROXY en ambos servidores.

```
systemctl start haproxy
```

## <img width="50" height="30" alt="image" src="https://github.com/user-attachments/assets/edaddf0d-4d6b-48ea-925b-577c55775b69" /> Prerrequisitos de sistema operativo:
####  Desactivar firewall del servidor.
 ```
 systemctl disable firewalld.service
 ```
####  Desactivar la swap
```
vim /etc/fstab
```
Comentamos la linea /dev/mapper/vg_system-lv_swap none

####  Cambiar SELINUX a modo permisivo para que los contenedores puedan acceder a los archivos del sistema. 
```
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```
####  El siguiente comando configura el reenvio de paquetes entre servidores.
```
cat <<EOF | tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```



## <img width="50" height="30" alt="image" src="https://github.com/user-attachments/assets/855930db-d4d5-4a73-ad64-ce272a187561" /> Instalacion del runtime, en este caso intalaremos containerd:

####  Desinstalar todo rastro de docker o otro runtime en el servidor.
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
####  Instalar algunos plugines necesarios.
```                  
sudo dnf -y install dnf-plugins-core
```
####  Configurar el repositorio  de docker para realizar la descarga de los paquetes.
```
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
```
#### Instalamos containerd
```
dnf install containerd
```
En la ruta /etc/containerd agregamos el archivo config.toml adjunto en este repo

####  Habilitamos el servicio para que inicie con el sistema opeartivo.
```
systemctl enable --now containerd
```



## <img width="50" height="30" alt="image" src="https://github.com/user-attachments/assets/8cde9c1c-b029-4b9b-9175-cc84217cafbe" /> Instalacion de kubernetes:

####  Configurar repositorio para descargar paquetes de Kubernetes
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
####  Instalar kubelet, kubectl y kubeadm
```
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```
Despues de ejecutar los pasos anteriores desactivar el proxy de los servidores (si tiene alguno activo) para porder arrancar el master.
    
####  Iniciar cluster nodo principal.
```
kubeadm init --control-plane-endpoint=10.140.3.191:6443 --pod-network-cidr "10.244.0.0/16
```
10.140.3.191 es la ip flotante que balancea los nodos ha-proxy

####  Unir los demas nodos master:
```
sudo kubeadm join 10.140.3.191:6443 --token pnyi2u.yu5lwwrdb5kfrkzv --discovery-token-ca-cert-hash sha256:07fc56078e3c5b5368fecd636baa4459b0d4a146faf9730283735ba7e85285c0  --control-plane --certificate-key c4f24d58b2557e8a24948d9c31206bb5e56e5bcc808fda361e6c39c37c337a03
```
El certificate-key lo obtenemos con el comando:
```
sudo kubeadm init phase upload-certs --upload-certs
```
####  Habilitar permisos para ejecucion de comandos kubectl. hacerlo en los 3 nodos master:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Nota: No dar permisos al usuario root para kubectl.

#### Unir nodos worker

```
kubeadm token create --print-join-command
```

```
kubeadm join 192.xxx.xx.xx:6443 --token abcdef.0123456789abcdef --discovery-token-ca-cert-hash sha256:526c5...
```

## <img width="70" height="30" alt="image" src="https://github.com/user-attachments/assets/309d0d15-e6a4-4460-8340-4437da98c294" /> Instalación del complemento de red:

####  Instalación calico.
```
sudo curl https://raw.githubusercontent.com/projectcalico/calico/v3.29.2/manifests/calico.yaml -O
```
```
vim calico.yaml
````
Buscar la sesion CALICO_IPV4POOL_CIDR e indicarl el CIDR utilizado para la instalación del cluster:

####  Aplicamos el archivo ya configurado.
```
kubectl apply -f calico.yaml
```

Al instalar el complemento de red los nodos deben cambiar de estado NoReady a Ready, verificar con el comando

```
kubectl get nodes
```





