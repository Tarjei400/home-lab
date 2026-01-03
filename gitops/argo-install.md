
# Kubernetes Home Lab
## Install first node:

```shell
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.32.10+k3s1 sh -s - --cluster-init --token k3sblog --tls-san 192.168.32.2
```
Install Kube VIP so that it is floating between nodes in cluster to provide API HA
We only need it for control plane, Services will be handled by MetalLB
```shell
sudo tee /var/lib/rancher/k3s/server/manifests/kube-vip.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: kube-vip
  namespace: kube-system
spec:
  serviceAccountName: kube-vip
  hostNetwork: true
  containers:
  - name: kube-vip
    image: ghcr.io/kube-vip/kube-vip:v1.0.3
    args:
      - manager
    env:
      - name: vip_arp
        value: "true"
      - name: cp_enable
        value: "true"
      - name: vip_interface
        value: "eno1"
      - name: address
        value: "192.168.32.2"
    securityContext:
      capabilities:
        add:
          - NET_ADMIN
          - NET_RAW
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube-vip
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kube-vip
rules:
- apiGroups: ["coordination.k8s.io"]
  resources: ["leases"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kube-vip
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kube-vip
subjects:
- kind: ServiceAccount
  name: kube-vip
  namespace: kube-system
EOF

```


## Join another node to the cluster:

```shell
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.32.10+k3s1 sh -s - server --server https://192.168.32.2:6443 --token k3sblog
```

## Install ArgoCD

```shell
helm repo add argo https://argoproj.github.io/argo-helm
```
```shell
kubectl create namespace argocd
```
```shell
helm install argocd argo/argo-cd \
  -n argocd \
  -f argo-values.yaml
```

## Apply bootstrap Applications
```shell
cd gitops/clusters/home/bootstrap/

kubectl apply -f home-root.yaml
```

## Nodes DNS

To prevent issues with network DNS comming from router or something else 
On each node do
```shell
echo -e "nameserver 1.1.1.1\nnameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```

## Reserved ips :
192.168.32.2 -Kube API handled by kube vip


## Accessing GCP Secrets

Create service account :

```shell
gcloud iam service-accounts create eso-gsm \
  --description="External Secrets Operator access to GSM" \
  --display-name="eso-gsm"
```

Create read permission
```shell
gcloud projects add-iam-policy-binding <PROJECT_ID> \
  --member="serviceAccount:eso-gsm@<PROJECT_ID>.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
  ```


Download key file
```shell
gcloud iam service-accounts keys create eso-gsm-key.json \
  --iam-account=eso-gsm@<PROJECT_ID>.iam.gserviceaccount.com
  
```

Create secret in `secrets` namespace for external secrets to use
```shell
kubectl create secret generic gsm-sa-key \
  --namespace secrets \
  --from-file=key.json=eso-gsm-key.json
```

Create service account :

```shell
gcloud iam service-accounts create eso-gsm \
  --description="External Secrets Operator access to GSM" \
  --display-name="eso-gsm"
```

Create read permission
```shell
gcloud projects add-iam-policy-binding techyon-393614 \
  --member="serviceAccount:eso-gsm@techyon-393614.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
  ```


Download 
```shell
gcloud iam service-accounts keys create eso-gsm-key.json \
  --iam-account=eso-gsm@techyon-393614.iam.gserviceaccount.com
```