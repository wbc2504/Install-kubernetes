# Install-kubernetes
Instalación de un cluster  de Kubernetes de manera manual

Prerrequisitos de sistema operativo:
1- systemctl status firewalld.service
2- vim /etc/fstab comentar la linea siguiente:
![image](https://github.com/user-attachments/assets/e0285564-cd85-4d0f-8a06-05c259487099)

3- sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config --> cambiar SELINUX a modo permisivo para que los contenedores puedan acceder a los archivos del sistema.
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
                  
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
