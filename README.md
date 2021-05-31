# Cluster API on vSphere #

These are the setup steps to use ClusterAPI (capi) on vSphere. We start off with a simple *kind* cluster, therefore both _kind_ and _Docker_ need to be pre-installed.

`$ kind create cluster`

This will create a simple Kubernetes cluster using containers. Next, download the *clusterctl* binary and use it to initialize the kind cluster for ClusterAPO. This _init_ steps adds the ClusterAPI bits to the kind cluster. This requires a bunch of ENV vars to be setup as wel as the clusterctl binary installed. See [mgmt_clusterctl_vsphere](./mgmt-clusterctl-vsphere).

```yaml
./mgmt-clusterctl-vsphere
Fetching providers
Skipping installing cert-manager as it is already installed
Installing Provider="cluster-api" Version="v0.3.17" TargetNamespace="capi-system"
Installing Provider="bootstrap-kubeadm" Version="v0.3.17" TargetNamespace="capi-kubeadm-bootstrap-system"
Installing Provider="control-plane-kubeadm" Version="v0.3.17" TargetNamespace="capi-kubeadm-control-plane-system"
Installing Provider="infrastructure-vsphere" Version="v0.7.7" TargetNamespace="capv-system"

Your management cluster has been initialized successfully!
 
You can now create your first workload cluster by running the following:

  clusterctl config cluster [name] --kubernetes-version [version] | kubectl apply -f -
```

Now you can create a workload cluster. Again, I have scripted it with loads of env vars. See [wkld_clusterctl_vsphere](./wkld-clusterctl-vsphere).

```yaml
./wkld-clusterctl-vsphere
cluster.cluster.x-k8s.io/vsphere-quickstart created
vspherecluster.infrastructure.cluster.x-k8s.io/vsphere-quickstart created
vspheremachinetemplate.infrastructure.cluster.x-k8s.io/vsphere-quickstart created
kubeadmcontrolplane.controlplane.cluster.x-k8s.io/vsphere-quickstart created
kubeadmconfigtemplate.bootstrap.cluster.x-k8s.io/vsphere-quickstart-md-0 created
machinedeployment.cluster.x-k8s.io/vsphere-quickstart-md-0 created
clusterresourceset.addons.cluster.x-k8s.io/vsphere-quickstart-crs-0 created
secret/vsphere-csi-controller created
configmap/vsphere-csi-controller-role created
configmap/vsphere-csi-controller-binding created
secret/csi-vsphere-config created
configmap/csi.vsphere.vmware.com created
configmap/vsphere-csi-node created
configmap/vsphere-csi-controller created
configmap/internal-feature-states.csi.vsphere.vmware.com created
```

At this point, you should see the VMs to back the K8s nodes be provisioned in vSphere. There will also be a cluster manifest created called _cluster.yaml_. Examine this to see the range of K8s objects that are created. This manifest can also be used to delete the cluster, but Cluster API also provides a more elegant way to do this which we will see shortly. 

Note that I had to create the folder in the vSphere inventory for this to happen (I think).


## KUBECONFIG

To retrieve the KUBECONFIG, do the following:

`$ clusterctl get kubeconfig vsphere-quickstart > quickstart-kubeconfig`

```
$ kubectl config get-contexts --kubeconfig quickstart-kubeconfig
CURRENT   NAME                                          CLUSTER              AUTHINFO                   NAMESPACE
*         vsphere-quickstart-admin@vsphere-quickstart   vsphere-quickstart   vsphere-quickstart-admin
```

## CNI 

Remember to apply a CNI or the nodes will not become READY.

`$ kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml --kubeconfig quickstart-kubeconfig`

```
$ kubectl get nodes --kubeconfig quickstart-kubeconfig  -o wide
NAME                                      STATUS   ROLES    AGE     VERSION            INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                 KERNEL-VERSION   CONTAINER-RUNTIME
vsphere-quickstart-795rm                  Ready    master   9m26s   v1.18.6+vmware.1   10.27.51.25   10.27.51.25   VMware Photon OS/Linux   4.19.132-1.ph3   containerd://1.3.4
vsphere-quickstart-md-0-57ff99b55-jtpg7   Ready    <none>   7m15s   v1.18.6+vmware.1   10.27.51.29   10.27.51.29   VMware Photon OS/Linux   4.19.132-1.ph3   containerd://1.3.4
vsphere-quickstart-md-0-57ff99b55-q9xd4   Ready    <none>   7m19s   v1.18.6+vmware.1   10.27.51.42   10.27.51.42   VMware Photon OS/Linux   4.19.132-1.ph3   containerd://1.3.4
vsphere-quickstart-md-0-57ff99b55-xd665   Ready    <none>   7m13s   v1.18.6+vmware.1   10.27.51.41   10.27.51.41   VMware Photon OS/Linux   4.19.132-1.ph3   containerd://1.3.4
```

## Cleanup

Start with the workload cluster, then remove the CAPI stuff fromt he management cluster, then finally delete the _kind_ cluster

```
$ kubectl delete cluster vsphere-quickstart
cluster.cluster.x-k8s.io "vsphere-quickstart" deleted
```
```
$ clusterctl delete --all
Deleting Provider="bootstrap-kubeadm" Version="v0.3.17" TargetNamespace="capi-kubeadm-bootstrap-system"
Deleting Provider="control-plane-kubeadm" Version="v0.3.17" TargetNamespace="capi-kubeadm-control-plane-system"
Deleting Provider="cluster-api" Version="v0.3.17" TargetNamespace="capi-system"
Deleting Provider="infrastructure-vsphere" Version="v0.7.7" TargetNamespace="capv-system"
```
```
$ kind delete cluster
Deleting cluster "kind" ...
```
