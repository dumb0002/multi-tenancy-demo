



## 1. Create a KIND Cluster with UDN CNI


```bash
git clone https://github.com/ovn-kubernetes/ovn-kubernetes.git
cd ovn-kubernetes
```

## 2. Build the Image for Fedora and Launch the KIND Deployment


```bash
pushd dist/images
make fedora-image
popd

pushd contrib
export KUBECONFIG=${HOME}/ovn.conf
./kind.sh
popd
```

### Check the Components

```bash
kubectl -n ovn-kubernetes get pods
# Expected output:
# NAME                                    READY   STATUS    RESTARTS      AGE
# ovnkube-control-plane-5b955978b-hwkjd   1/1     Running   1 (33m ago)   33m
# ovnkube-identity-bmn6m                  1/1     Running   0             33m
# ovnkube-node-gnvlg                      6/6     Running   0             33m
# ovnkube-node-p2nxh                      6/6     Running   0             33m
# ovnkube-node-rhr6p                      6/6     Running   0             33m
# ovs-node-9mxqt                          1/1     Running   0             33m
# ovs-node-tntpk                          1/1     Running   0             33m
# ovs-node-wh2nb                          1/1     Running   0             33m
```

## 3. Deploy KubeVirt

```bash
export VERSION=$(curl -s https://storage.googleapis.com/kubevirt-prow/release/kubevirt/kubevirt/stable.txt)

echo $VERSION
kubectl create -f "https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/kubevirt-operator.yaml"
```

### Deploy the KubeVirt Custom Resource Definitions

```bash
kubectl create -f "https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/kubevirt-cr.yaml"
```

### Enable KubeVirt Emulation

```bash
kubectl -n kubevirt patch kubevirt kubevirt --type=merge --patch '{"spec":{"configuration":{"developerConfiguration":{"useEmulation":true}}}}'
```

### Check the Components

```bash
kubectl get all -n kubevirt
```

```console
# Expected output:
# virt-api-7cc64fb85f-4pdpt         1/1     Running   0          31s
# virt-api-7cc64fb85f-tjbgv         1/1     Running   0          113s
# virt-controller-964c8dbcb-6kxn2   1/1     Running   0          85s
# virt-controller-964c8dbcb-p82bg   1/1     Running   0          85s
# virt-handler-fmqtc                1/1     Running   0          85s
# virt-handler-q49zq                1/1     Running   0          85s
# virt-handler-zfwf9                1/1     Running   0          85s
# virt-operator-68f69776bc-dx97s    1/1     Running   0          8m8s
# virt-operator-68f69776bc-wdpw2    1/1     Running   0          8m8s
```

## 4. Deploy KubeFlex

### Install CLI

```bash
sudo su <<EOF
bash <(curl -s https://raw.githubusercontent.com/kubestellar/kubeflex/main/scripts/install-kubeflex.sh) --ensure-folder /usr/local/bin --strip-bin
EOF
```

### Deploy KubeFlex Controller

```bash
kubectl create ns kubeflex-system
helm upgrade --install kubeflex-operator oci://ghcr.io/kubestellar/kubeflex/chart/kubeflex-operator \
  --version v0.9.3 \
  --namespace kubeflex-system \
  --set domain=localtest.me \
  --set externalPort=9443
```

### Check the Components

```bash
kubectl -n kubeflex-system get pods
```

```console
# Expected output:
# NAME                                           READY   STATUS      RESTARTS   AGE
# kubeflex-controller-manager-6fdf485568-mfzxf   2/2     Running     0          54s
# kubeflex-operator-ffjn2                        0/1     Completed   0          54s
# postgres-postgresql-0                          1/1     Running     0          48s
```

## 5. Create K3s Control Plane for Tenant-1 & Tenant-2

```bash
kflex create tenant-1-cp --type k3s --kubeconfig=/root/ovn.conf
```

### Check the Components

```bash
kubectl config get-contexts
```

```console
# Expected output:
# CURRENT   NAME          CLUSTER               AUTHINFO            NAMESPACE
#           kind-ovn      kind-ovn              kind-ovn
# *         tenant-1-cp   tenant-1-cp-cluster   tenant-1-cp-admin
```

```bash
kubectl config use-context kind-ovn

kubectl -n tenant-1-cp-system get pods
```

```console
# Expected output:
# NAME                             READY   STATUS      RESTARTS   AGE
# k3s-bootstrap-kubeconfig-s2llj   0/2     Completed   4          3m33s
# k3s-server-0                     1/1     Running     0          3m33s
```

### Patch the K3s StatefulSet for UDN Configurations

#### 1. Apply the Patch

```bash
kubectl -n tenant-1-cp-system patch statefulset k3s-server --type=strategic --patch-file k3s-patch.yaml
```

#### 2. Verify the Patch

```bash
# Check if patch was applied
kubectl -n tenant-1-cp-system get statefulset k3s-server -o yaml | grep -A 5 "k8s.v1.cni.cncf.io/networks"

# Watch pod restart
kubectl -n tenant-1-cp-system get pods -w

# Check pod logs after restart
kubectl -n tenant-1-cp-system logs k3s-server-0 -f

# Verify nodes
kubectl -n tenant-1-cp-system exec k3s-server-0 -- kubectl get nodes -o wide
```

```console
# Expected output:
# NAME           STATUS     ROLES                  AGE   VERSION         INTERNAL-IP   EXTERNAL-IP   OS-IMAGE            KERNEL-VERSION    CONTAINER-RUNTIME
# k3s-server-0   NotReady   control-plane,master   17m   v1.30.13+k3s1   104.104.0.3   104.104.0.3   K3s v1.30.13+k3s1   6.14.0-1011-aws   containerd://1.7.27-k3s1
```

### 3. Create VM and Attach to K3s Control Plane

#### a) Extract Join Token

```bash
kubectl exec -n tenant-1-cp-system k3s-server-0 -- cat /var/lib/rancher/k3s/server/node-token
```
