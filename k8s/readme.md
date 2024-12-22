#### References:
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/

https://www.server-world.info/en/note?os=Ubuntu_24.04&p=kubernetes&f=2

https://hbayraktar.medium.com/how-to-install-kubernetes-cluster-on-ubuntu-22-04-step-by-step-guide-7dbf7e8f5f99

#### In all the nodes:
```
apt-get install -y apt-transport-https ca-certificates curl gpg
rm -f /etc/apt/keyrings/kubernetes-apt-keyring.gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubelet kubeadm kubectl containerd
apt-mark hold kubelet kubeadm kubectl
systemctl enable --now kubelet

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

systemctl restart containerd.service
```

### in first master node:
```
kubeadm init --control-plane-endpoint=LOAD_BALANCER_DNS:LOAD_BALANCER_PORT --apiserver-advertise-address=LOCALNODE_IP_TO_ADVERTISE  --pod-network-cidr=NON_CONFLICTING_CIDR --cri-socket=unix:///run/containerd/containerd.sock
```
For example:
```
kubeadm init --control-plane-endpoint=k8s.example.internal:6443 --apiserver-advertise-address=192.168.200.21 --pod-network-cidr=10.128.0.0/14 --cri-socket=unix:///run/containerd/containerd.sock
```

### install calico:
```
helm repo add projectcalico https://docs.tigera.io/calico/charts
helm install calico projectcalico/tigera-operator --version v3.29.1 --namespace tigera-operator
```

### in second and third master node:
```
kubeadm join endpoint=k8s.example.internal:6443 --token TOKEN_DISPLAYED_IN_INIT \
        --discovery-token-ca-cert-hash CA_CERT_HASH_DISPLAYED_IN_INIT \
        --control-plane
```

All nodes should be in ready status:
```
k get nodes -o wide
NAME   STATUS   ROLES           AGE     VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
k01    Ready    control-plane   29m     v1.31.4   192.168.200.21   <none>        Ubuntu 24.04.1 LTS   6.8.0-49-generic   containerd://1.7.12
k02    Ready    control-plane   16m     v1.31.4   192.168.200.22   <none>        Ubuntu 24.04.1 LTS   6.8.0-51-generic   containerd://1.7.12
k03    Ready    control-plane   8m14s   v1.31.4   192.168.200.23   <none>        Ubuntu 24.04.1 LTS   6.8.0-51-generic   containerd://1.7.12
root@k01:~# 
```

### join worker nodes:
```
kubeadm join k8s.example.internal:6443 --token 7fx3db.gxvq3ziy21xhqh46 \
        --discovery-token-ca-cert-hash sha256:8430b2820ee4bc0d5f89c62039c90d942eaa8323b5d112a97213ea7388147831
```

### remove a worker node:
```
kubectl cordon NODE # do not accept any moer pods
kubectl drain NODE --ignore-daemonsets --delete-emptydir-data
kubectl delete node NODE

#on the node:
kubeadm reset -f
rm -rf ~/.kube /etc/cni /etc/kubernetes /var/lib/etcd /var/lib/kubelet
iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && iptables -X
```



## Test deployment:
```
kubectl create deployment test-nginx --image=nginx
kubectl get pods
POD_NAME=$(kubectl get pods -l app=test-nginx -o custom-columns=":metadata.name" | tail -n 1)
echo Pod name: $POD_NAME
APP_NAME=$(kubectl get pods -l app=test-nginx -o custom-columns=":metadata.labels.app" | tail -n 1)
echo Application name: $APP_NAME



kubectl exec ${POD_NAME} -- env
kubectl exec -it ${POD_NAME} -- bash
      hostname
      exit
kubectl logs ${POD_NAME}
kubectl scale deployment test-nginx --replicas=3
kubectl get pods -o wide
kubectl logs -l app=test-nginx -f


kubectl expose deployment test-nginx --type="NodePort" --port 80
kubectl get services test-nginx
kubectl port-forward service/test-nginx --address 127.0.0.1 8082:80 &
curl localhost:8082
kubectl delete services test-nginx
kubectl delete deployment test-nginx 
```


## helm, basic usage
```
# Optional snap install helm 
curl -O https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 
bash ./get-helm-3
helm search hub wordpress 
helm repo add bitnami https://charts.bitnami.com/bitnami 
helm repo list 
helm search repo bitnami 
helm show chart bitnami/haproxy 
helm install haproxy bitnami/haproxy 
helm list 
helm status haproxy 
helm uninstall haproxy
```

## dashboard
```
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
helm install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard

cat <<EOF > rbac.yml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF

kubectl apply -f rbac.yml

kubectl -n kubernetes-dashboard create serviceaccount admin-user
kubectl -n kubernetes-dashboard create token admin-user
kubectl port-forward -n kubernetes-dashboard svc/kubernetes-dashboard-kong-proxy --address 0.0.0.0 8443:443  &


```









