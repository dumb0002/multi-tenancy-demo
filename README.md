# Three Shades of Isolation: A Multi-tenancy Fortress


## 1. Create a KIND Cluster with UDN CNI


```bash
git clone https://github.com/ovn-kubernetes/ovn-kubernetes.git
cd ovn-kubernetes
```

## 2. Build the Image for Fedora and Launch the KIND Deployment

Pull and tag pre-built image: 
```bash
docker pull ghcr.io/ovn-org/ovn-kubernetes/ovn-kube-fedora:master
docker tag ghcr.io/ovn-org/ovn-kubernetes/ovn-kube-fedora:master ovn-daemonset-fedora
```

Continue with KIND deployment: 
```bash
pushd contrib
export KUBECONFIG=${HOME}/ovn.conf
./kind.sh -mne -nse
popd
```

Check the Components:

```bash
kubectl -n ovn-kubernetes get pods
```

Expected similar output:
```console
NAME                                    READY   STATUS    RESTARTS       AGE
ovnkube-control-plane-5b955978b-t922v   1/1     Running   1 (114s ago)   2m6s
ovnkube-identity-r225g                  1/1     Running   0              2m7s
ovnkube-node-nm5p5                      6/6     Running   0              2m5s
ovnkube-node-wm9rj                      6/6     Running   0              2m5s
ovnkube-node-x8cmc                      6/6     Running   0              2m5s
ovs-node-2mhjv                          1/1     Running   0              2m8s
ovs-node-2z99g                          1/1     Running   0              2m8s
ovs-node-7p7fz                          1/1     Running   0              2m8s
```

## 3. Deploy KubeVirt

```bash
export VERSION=$(curl -s https://storage.googleapis.com/kubevirt-prow/release/kubevirt/kubevirt/stable.txt)

echo $VERSION
kubectl create -f "https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/kubevirt-operator.yaml"
```

Deploy the KubeVirt Custom Resource Definitions:

```bash
kubectl create -f "https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/kubevirt-cr.yaml"
```

Enable KubeVirt Emulation:

```bash
kubectl -n kubevirt patch kubevirt kubevirt --type=merge --patch '{"spec":{"configuration":{"developerConfiguration":{"useEmulation":true}}}}'
```

Check the Components:

```bash
kubectl get all -n kubevirt
```

Expected similar output:
```console
NAME                               READY   STATUS    RESTARTS   AGE
virt-api-9d99b8957-46tkc           1/1     Running   0          2m47s
virt-api-9d99b8957-g7kwr           1/1     Running   0          85s
virt-controller-569dcc495f-hgw5l   1/1     Running   0          2m21s
virt-controller-569dcc495f-vslnh   1/1     Running   0          2m21s
virt-handler-78nhb                 1/1     Running   0          2m21s
virt-handler-j5sbd                 1/1     Running   0          2m21s
virt-handler-jlk74                 1/1     Running   0          2m21s
virt-operator-7c4dc9d77f-4m7dm     1/1     Running   0          3m22s
virt-operator-7c4dc9d77f-jzrh2     1/1     Running   0          3m22s
```

## 4. Deploy KubeFlex

Install CLI:

```bash
sudo su <<EOF
bash <(curl -s https://raw.githubusercontent.com/kubestellar/kubeflex/main/scripts/install-kubeflex.sh) --ensure-folder /usr/local/bin --strip-bin
EOF
```

Deploy KubeFlex Controller:

```bash
kubectl create ns kubeflex-system
helm upgrade --install kubeflex-operator oci://ghcr.io/kubestellar/kubeflex/chart/kubeflex-operator \
  --version v0.9.3 \
  --namespace kubeflex-system \
  --set domain=localtest.me \
  --set externalPort=9443
```

Check the Components:

```bash
kubectl -n kubeflex-system get pods
```

Expected similar output:
```console
NAME                                           READY   STATUS      RESTARTS   AGE
kubeflex-controller-manager-6fdf485568-xzp2t   2/2     Running     0          2m3s
kubeflex-operator-g76mk                        0/1     Completed   0          2m3s
postgres-postgresql-0                          1/1     Running     0          109s
```

## 5. Create UDNs for Tenant-1 & Tenant-2


```bash
kubectl create -f udn-tenant-1.yaml
```

Check the Components:

```bash
kubectl create -f udn-tenant-1.yaml
```

```bash
kubectl get udn,cudn -A
```

Expected output:
```console
NAMESPACE   NAME                                         AGE
tenant-1    userdefinednetwork.k8s.ovn.org/tenant-1-dp   3m36s

NAMESPACE   NAME                                                      AGE
            clusteruserdefinednetwork.k8s.ovn.org/tenant-1-cp         3m36s
            clusteruserdefinednetwork.k8s.ovn.org/tenant-1-external   3m36s
```

Check 

```bash
kubectl get cudn tenant-1-cp -o jsonpath='{.status.conditions}' | jq
```

Expected similar output:

```console
[
  {
    "lastTransitionTime": "2026-04-13T18:59:58Z",
    "message": "NetworkAttachmentDefinition has been created in following namespaces: [tenant-1]",
    "reason": "NetworkAttachmentDefinitionCreated",
    "status": "True",
    "type": "NetworkCreated"
  }
]
```


## 6. Create K3s Control Plane for Tenant-1 & Tenant-2

```bash
kflex create tenant-1-cp --type k3s --kubeconfig=/root/ovn.conf
```

Check the Components:

```bash
kubectl config get-contexts
```

Expected similar output:
```console
CURRENT   NAME          CLUSTER               AUTHINFO            NAMESPACE
          kind-ovn      kind-ovn              kind-ovn            
*         tenant-1-cp   tenant-1-cp-cluster   tenant-1-cp-admin 
```

```bash
kubectl config use-context kind-ovn
kubectl -n tenant-1-cp-system get pods
```

Expected similar output:
```console
NAME                             READY   STATUS      RESTARTS   AGE
k3s-bootstrap-kubeconfig-pfm2r   0/2     Completed   4          92s
k3s-server-0                     1/1     Running     0          92s
```

Patch the K3s StatefulSet for UDN Configurations:

1. Apply the Patch:

```bash
kubectl -n tenant-1-cp-system patch statefulset k3s-server --type=strategic --patch-file k3s-patch.yaml
```

2. Verify the Patch:

Check if patch was applied and the k3s control pod has joined the UDN control plane network:
```bash
kubectl -n tenant-1-cp-system get pod k3s-server-0 -o jsonpath='{.metadata.annotations.k8s\.v1\.cni\.cncf\.io/network-status}'
```

Expected similar output:
```console
[{
    "name": "ovn-kubernetes",
    "interface": "eth0",
    "ips": [
        "10.244.1.13"
    ],
    "mac": "0a:58:0a:f4:01:0d",
    "default": true,
    "dns": {}
},{
    "name": "tenant-1-cp-system/tenant-1-cp",
    "interface": "net1",
    "ips": [
        "104.104.0.2"
    ],
    "mac": "0a:58:68:68:00:02",
    "dns": {}
}]
```

Verify nodes
```bash
kubectl -n tenant-1-cp-system exec k3s-server-0 -- kubectl get nodes -o wide
```

Expected similar output:
```console
NAME           STATUS   ROLES                  AGE   VERSION         INTERNAL-IP   EXTERNAL-IP   OS-IMAGE            KERNEL-VERSION    CONTAINER-RUNTIME
k3s-server-0   Ready    control-plane,master   26m   v1.30.13+k3s1   104.104.0.2   104.104.0.2   K3s v1.30.13+k3s1   6.14.0-1011-aws   containerd://1.7.27-k3s1
```

## 7. Create VM and Attach to K3s Control Plane

#### a) Extract Join Token

```bash
kubectl exec -n tenant-1-cp-system k3s-server-0 -- cat /var/lib/rancher/k3s/server/node-token
```
