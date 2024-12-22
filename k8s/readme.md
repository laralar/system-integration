#https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/
##https://www.server-world.info/en/note?os=Ubuntu_24.04&p=kubernetes&f=2
#https://hbayraktar.medium.com/how-to-install-kubernetes-cluster-on-ubuntu-22-04-step-by-step-guide-7dbf7e8f5f99

apt-get install -y apt-transport-https ca-certificates curl gpg
rm -f /etc/apt/keyrings/kubernetes-apt-keyring.gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
systemctl enable --now kubelet
apt -y install containerd
mkdir /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
sed -e "s/sandbox_image.*/sandbox_image = \"registry.k8s.io\/pause:3.9\"/" -i  /etc/containerd/config.toml
cat > /etc/sysctl.d/99-k8s-cri.conf <<EOF
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF
sysctl --system

swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
echo -e overlay\\nbr_netfilter > /etc/modules-load.d/k8s.conf
modprobe overlay; modprobe br_netfilter
update-alternatives --set iptables /usr/sbin/iptables-legacy
apparmor_parser -R /etc/apparmor.d/runc
apparmor_parser -R /etc/apparmor.d/crun
ln -s /etc/apparmor.d/runc /etc/apparmor.d/disable/
ln -s /etc/apparmor.d/crun /etc/apparmor.d/disable/

apt update -y
apt upgrade -y
apt autoremove -y
