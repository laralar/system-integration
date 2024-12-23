#### References:
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/

https://www.server-world.info/en/note?os=Ubuntu_24.04&p=kubernetes&f=2

https://hbayraktar.medium.com/how-to-install-kubernetes-cluster-on-ubuntu-22-04-step-by-step-guide-7dbf7e8f5f99

https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart

# Basic setup
#### In all the nodes:
Prepare the node for a deployment (control-plane or worker node)

```
apt-get install -y apt-transport-https ca-certificates curl gpg
rm -f /etc/apt/keyrings/kubernetes-apt-keyring.gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubelet kubeadm kubectl containerd
apt-mark hold kubelet kubeadm kubectl
apt upgrade -y
apt autoremove -y
systemctl enable --now kubelet

mkdir -p /etc/containerd
mkdir -p /etc/kubernetes/manifests
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

systemctl restart containerd.service
```

## in the first master node:

Let's specify the pod network and service network, this will be used for assigning IP to pods, /16 is enough for a small deployment.   

If the node has multiple interfaces, it is better to also specify the api-server address; so that it is published in the correct interface

```
kubeadm init --control-plane-endpoint=LOAD_BALANCER_FQDN:LOAD_BALANCER_PORT --apiserver-advertise-address=NODE_IP --service-cidr=10.96.0.0/16 --pod-network-cidr=10.128.0.0/16 --cri-socket=unix:///run/containerd/containerd.sock --upload-certs
```
For example:
```
kubeadm init --control-plane-endpoint=k8s.cloud.example.org:6443 --apiserver-advertise-address=192.168.200.21 --service-cidr=10.96.0.0/16 --pod-network-cidr=10.128.0.0/16 --cri-socket=unix:///run/containerd/containerd.sock --upload-certs
```

### install calico:
calico is used for networking, other optiosn available.  by default each node has an IP that is accessible from the host. (pod network)  if we want pods in one host to communicate with pods in other hosts, they have to be able to reach the other pod; calico enables that inter pod, inter host communication, routing traffic through the hosts physical networks.

By default calico autodiscovers the network cidr where that it is going to use in the nodes; this is fine if the node has only one network interface; if there are multiple interfaces we have to specify the autodetection cidr to avoid the selection of a network that would make pod to pod communicationunreachable

```
cat <<EOF > calico.yml
# This section includes base Calico installation configuration.
# For more information, see: https://docs.tigera.io/calico/latest/reference/installation/api#operator.tigera.io/v1.Installation
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # Configures Calico networking.
  calicoNetwork:
    ipPools:
    - name: default-ipv4-ippool
      blockSize: 26
      cidr:  10.128.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
    nodeAddressAutodetectionV4:
      cidrs:
        - '192.168.200.0/24'

---

# This section configures the Calico API server.
# For more information, see: https://docs.tigera.io/calico/latest/reference/installation/api#operator.tigera.io/v1.APIServer
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
EOF
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/tigera-operator.yaml
kubectl apply -f calico.yaml

```

## in second and third master node:
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

## join worker nodes:
```
kubeadm join k8s.example.internal:6443 --token 7fx3db.gxvq3ziy21xhqh46 \
        --discovery-token-ca-cert-hash sha256:8430b2820ee4bc0d5f89c62039c90d942eaa8323b5d112a97213ea7388147831
```

## remove a node:
```
kubectl cordon NODE # do not accept any moer pods
kubectl drain NODE --ignore-daemonsets --delete-emptydir-data
kubectl delete node NODE
```

### cleanup the node for another deployment
```
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

#nslookup can test if pods in other nodes can reach the coredns pod; I had issues where I could not because calico was using the wrong network
kubectl run test-pod --image=busybox --restart=Never -- sleep 3600
kubectl exec -it test-pod -- nslookup kubernetes.default.svc.cluster.local


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

## load balancer
```
helm repo add metallb https://metallb.github.io/metallb --force-update

helm install metallb metallb/metallb -n metallb-system --create-namespace
cat <<EOF > metallb-config.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: cheap
  namespace: metallb-system
spec:
  addresses:
  - 192.168.200.60-192.168.200.99
EOF
kubectl apply -f metallb-config.yaml


```

## ingress
```


helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx --force-update
helm show values ingress-nginx/ingress-nginx | tee defaultValues-ingress-nginx.yzml

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.watchIngressWithoutClass=true \
  --set controller.allowSnippetAnnotations=true

or

helm install ingress-nginx  ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace -f values-ingress-nginx.yaml


kubectl scale deployment ingress-nginx-controller -n ingress-nginx --replicas=3
```

## dashboard (optional)
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

## external ceph storage
Create sample deployment files
```
helm repo add ceph-csi https://ceph.github.io/csi-charts --force-update
helm show values ceph-csi/ceph-csi-rbd |tee defaultValues-ceph-csi-rbd.yaml
helm show values ceph-csi/ceph-csi-cephfs |tee defaultValues-ceph-csi-cephfs.yaml
```

```
cat <<EOF > values-ceph-csi-rbd.yaml
cephConfConfigMapName: ceph-config-rbd
configMapName: ceph-csi-config-rbd
csiConfig:
   - clusterID: "<cluster-id>"
     monitors:
       - "<MONValue1:PORT>"
       - "<MONValue2:PORT>"
storageClass:
  clusterID: "<cluster-id>"
  create: true
  # Needed if using a specific data pool (e.g. erasure codeing pool)
  dataPool: ''
  pool: kubernetes
EOF
cat <<EOF > values-ceph-csi-cephfs.yaml
cephConfConfigMapName: ceph-config-cephfs
configMapName: ceph-csi-config-cephfs
csiConfig:
   - clusterID: "<cluster-id>"
     monitors:
       - "<MONValue1:PORT>"
       - "<MONValue2:PORT>"
    cephFS:
      # (optional) CephFS sub volume group
      subvolumeGroup: "csi"
storageClass:
  create: true
  name: csi-cephfs-sc
  clusterID: <CLUSTER-ID>
  # (required) CephFS filesystem name into which the volume shall be created
  fsName: cephfs
  reclaimPolicy: Delete
  allowVolumeExpansion: true
  # (optional) CephFS sub volume prefix
  volumeNamePrefix: "poc-k8s-"
  provisionerSecret: csi-cephfs-secret
  controllerExpandSecret: csi-cephfs-secret
  nodeStageSecret: csi-cephfs-secret
  pool: ''
EOF

cat <<EOF > csi-rbd-secret.yaml
kind: Secret
apiVersion: v1
metadata:
  name: csi-rbd-secret
  namespace: ceph-csi-rbd
data:
  userID: BASE64_USERID
  userKey: BASE64_KEY
type: Opaque
EOF
cat <<EOF > csi-cephfs-secret.yaml
kind: Secret
apiVersion: v1
metadata:
  name: csi-cephfs-secret
  namespace: ceph-csi-cephfs
data:
  adminID: BASE64_ADMIN_USERID
  adminKey: BASE64_ADMIN_KEY
type: Opaque
EOF
kubectl apply -f csi-rbd-secret.yaml
kubectl apply -f csi-cephfs-secret.yaml

helm upgrade --install ceph-csi-rbd ceph-csi/ceph-csi-rbd --values values-ceph-csi-rbd.yaml -n ceph-csi-rbd --create-namespace
helm upgrade --install ceph-csi-cephfs ceph-csi/ceph-csi-cephfs --values values-ceph-csi-cephfs.yaml  -n ceph-csi-cephfs --create-namespace
kubectl patch storageclass csi-rbd-sc -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "true"}}}'

cat <<EOF > raw-block-pvc-.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: image-registry-storage
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Block
  resources:
    requests:
      storage: 1Ti
  storageClassName: csi-rbd-sc
EOF
kubectl apply -f raw-block-pvc-.yaml
kubectl get pvc
# it has to be in bound state

```

## kubevirt (optional)
```
export VERSION=$(curl https://storage.googleapis.com/kubevirt-prow/release/kubevirt/kubevirt/stable.txt)
wget https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/kubevirt-operator.yaml
wget https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/kubevirt-cr.yaml
wget https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/virtctl-${VERSION}-linux-amd64
mv virtctl-${VERSION}-linux-amd64 /usr/local/bin/virtctl
chmod 755 /usr/local/bin/virtctl
kubectl apply -f kubevirt-operator.yaml
kubectl apply -f kubevirt-cr.yaml
kubectl get pods -n kubevirt

wget https://raw.githubusercontent.com/kubevirt/kubevirt.github.io/master/labs/manifests/vm.yaml
kubectl apply -f vm.yaml
kubectl get vms
virtctl start testvm
kubectl get vms
kubectl get vmi
virtctl console testvm
kubectl get pods
kubectl port-forward pod/virt-launcher-testvm-fkccd 222:22 &
ssh cirros@localhost -p 222
virtctl stop testvm
kubectl get vms
kubectl delete vm testvm
kubectl get vms
```












