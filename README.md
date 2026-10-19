--Sistema Base
sudo dnf update -y
sudo dnf install -y curl wget tar iptables container-selinux epel-release util-linux net-tools

sudo dnf install -y kernel-modules-extra-$(uname -r)
sudo dnf install -y iptables
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -P FORWARD ACCEPT
sudo iptables -F FORWARD
sudo dnf install -y ethtool
sudo ethtool -K flannel.1 tx off
sudo dnf install -y tcpdump

--Instalacion del Cluster, Todos los nodos


sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

cat <<EOF | sudo tee /etc/modules-load.d/k3s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/99-k3s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

cat <<EOF | sudo tee /etc/NetworkManager/conf.d/k3s.conf
[keyfile]
unmanaged-devices=interface-name:flannel*;interface-name:cni0
EOF

sudo systemctl reload NetworkManager

sudo systemctl disable --now firewalld
# Desactivar aceleración de checksum en la tarjeta física
sudo ethtool -K enp2s0 rx off tx off

# Desactivar rp_filter en todas las interfaces
for f in /proc/sys/net/ipv4/conf/*/rp_filter; do echo 0 | sudo tee "$f" > /dev/null; done

# Permitir reenvío en iptables
sudo iptables -P FORWARD ACCEPT


sudo dnf install -y kernel-modules-extra-$(uname -r) kernel-modules-extra


# NOMBRAR AL NODO MASTER
sudo hostnamectl set-hostname srv-k3s-master
sudo systemctl restart k3s

sudo sysctl --system

-- Nodo Master
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server \
  --tls-san=ecria.aseguradoradelsur.xyz \
  --tls-san=172.18.0.21 \
  --write-kubeconfig-mode=644" sh -

--Verificar el estado del Master:
kubectl get nodes


# Abrir el puerto permanentemente
sudo firewall-cmd --permanent --zone=trusted --add-source=10.42.0.0/16
sudo firewall-cmd --permanent --zone=trusted --add-source=10.43.0.0/16
sudo firewall-cmd --permanent --add-port=8472/udp
sudo firewall-cmd --permanent --add-port=9443/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --reload


--Obtener el Token de unión para los Workers:
sudo cat /var/lib/rancher/k3s/server/node-token


--Unir los Nodos Workers al Clúster

# NOMBRAR AL NODO WORKER
sudo hostnamectl set-hostname srv-k3s-worker-01

curl -sfL https://get.k3s.io | K3S_URL=https://192.168.217.135:6443 \
  K3S_TOKEN="K104b05033dabd3fb05fda0b5244041f695dde9d6a941fe2acae9582a3d96896b1e::server:31f81b9a34a8bee0b87628df5b112d6d" sh -


--En caso de error para ver los logs
sudo journalctl -u k3s-agent -n 10 --no-pager



# 1. NOMBRAR AL NODO DATA
sudo hostnamectl set-hostname srv-k3s-data-01

SALIR $ INGRESAR

# 2. Unir al clúster
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.217.135:6443 \
  K3S_TOKEN="K104b05033dabd3fb05fda0b5244041f695dde9d6a941fe2acae9582a3d96896b1e::server:31f81b9a34a8bee0b87628df5b112d6d" sh -


# 1. NOMBRAR AL NODO VECTOR
sudo hostnamectl set-hostname srv-k3s-vector-01

# 2. Unir al clúster
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.217.135:6443 \
  K3S_TOKEN="K104b05033dabd3fb05fda0b5244041f695dde9d6a941fe2acae9582a3d96896b1e::server:31f81b9a34a8bee0b87628df5b112d6d" sh -


# TODOS LOS NODOS

sudo firewall-cmd --permanent --zone=trusted --add-source=10.42.0.0/16
sudo firewall-cmd --permanent --zone=trusted --add-source=10.43.0.0/16
sudo firewall-cmd --permanent --add-port=8472/udp
sudo firewall-cmd --permanent --add-port=9443/tcp
sudo firewall-cmd --reload

sudo ethtool -K enp2s0 rx off tx off
sudo sysctl -w net.ipv4.conf.all.rp_filter=0
sudo sysctl -w net.ipv4.conf.default.rp_filter=0
sudo sysctl -w net.ipv4.conf.enp2s0.rp_filter=0
echo 0 | sudo tee /proc/sys/net/ipv4/conf/flannel.1/rp_filter

sudo nft flush ruleset
sudo iptables -P FORWARD ACCEPT
sudo iptables -F FORWARD

--Etiquetado de Nodos
kubectl get nodes



--Aplicar etiquetas por rol de carga:
# Etiquetar nodo de cómputo para workers de n8n y gateway
kubectl label node srv-k3s-worker-01 workload=apps

# Etiquetar nodo de datos transaccionales (Postgres n8n y Redis)
kubectl label node srv-k3s-data-01 workload=data storage=fast

# Etiquetar nodo dedicado al motor vectorial pgvector (NVMe)
kubectl label node srv-k3s-vector-01 workload=vector storage=nvme


--Verificación final del Clúster
kubectl get nodes -o wide --show-labels


ssh root@192.168.217.135
ssh root@192.168.217.136

# DESPLIEGUE DE LA APLICACION

Paso 1: Instalar el operador CloudNativePG (Prerrequisito), 

kubectl apply --server-side -f \
  https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.24/releases/cnpg-1.24.0.yaml

Comprobar
# kubectl get deployment -n cnpg-system cnpg-controller-manager

# Paso 2: Crear los Namespaces
kubectl apply -f n8n-k3s-production/00-namespaces.yaml

# Paso 3: Aplicar las Network Policies de seguridad
kubectl apply -f n8n-k3s-production/01-security/

# Paso 4: Desplegar la capa de persistencia (data-state)
kubectl apply -f n8n-k3s-production/02-data-state/


kubectl apply -f n8n-k3s-production/04-gateway-dmz/

# Paso 6: Desplegar la capa de persistencia (data-state)
kubectl apply -f n8n-k3s-production/03-n8n-system/





