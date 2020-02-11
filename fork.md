# k3OS baremetal
This fork of the upstream k3OS add as default
- [MetalLB layer 2](https://metallb.universe.tf/concepts/layer2/) as default load-balancer
- [OpenEBS.io](https://github.com/openebs/openebs) with [cStor](https://github.com/openebs/cstor) as storage provider
- Helm 3 (without Tiller) with RBAC to support Helm-Chart package deployments
- Preconfigure using response/config-file within the iso

Please see [k3OS README](README.md) for [installation](https://ahmermansoor.blogspot.com/2019/05/install-lightweight-kubernetes-k3s-with-k3os.html):
- After boot the live ISO, login with user "rancher" (no password)
- Do "sudo passwd rancher" to set a default password

Readings
- [Deploy k3os and openebs](https://medium.com/@fromprasath/deploy-k3s-cluster-on-k3os-and-use-openebs-as-persistent-storage-provisioner-3db229c0acf8)
- [K3OS with MetalLB and Dashboad](https://mindmelt.nl/mindmelt.nl/2019/04/08/k3s-kubernetes-dashboard-load-balancer/)

# Connect to K8s api
Connect to your master-node using ssh.
- Check the cluster 
```
myhostname [~]$ kubectl get nodes
NAME         STATUS   ROLES    AGE   VERSION
myhostname   Ready    master   11m   v1.17.2+k3s1
```

- Type 
```
myhostname [~]$ kubectl config view --minify
          apiVersion: v1
          clusters:
          - cluster:
              certificate-authority-data: ...
              server: https://127.0.0.1:6443
            name: default
         ...
          users:
          - name: default
            user:
              password: 3289efrnke472ljf
              username: admin
```
- Connect using your browser ```https://ip-of-the-master-node:6443``` and ```user: admin, password: 3289efrnke472ljf``` to the K8s master-api
- List the kubernetes resources ```kubectl get svc,deployments,pods,ingresses -n kube-system``` or endpoints ```kubectl get endpoints -n kube-system```
- Read the logs of K3s ```sudo more /var/log/k3s-service.log```
- Get K8s cluster infos: 
```
CLUSTER_ENDPOINT=$(kubectl config view --raw -o json | jq -r '.clusters[] | select(.name == "'$(kubectl config current-context)'") | .cluster."server"')
CLUSTER_CA=$(kubectl config view --raw -o json | jq -r '.clusters[] | select(.name == "'$(kubectl config current-context)'") | .cluster."certificate-authority-data"')
CLIENT_CERTIFICATE_DATA=$(kubectl get csr mycsr -n kube-system -o jsonpath='{.status.certificate}')
CLUSTER_NAME=$(kubectl config view --minify -o jsonpath={.current-context})
ADMINUSER_TOKEN=$(kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}'))
echo $ADMINUSER_TOKEN | base64 -d > ./k3s_token
echo $CLUSTER_CA | base64 -d > ./k3s_ca
```
- Copy file ```k3s_token``` and ```k3s_ca``` using scp to your local computer 
- From within your local computer, add the cluster to your local kubectl client: 
```
kubectl config set-cluster myk3scluster --server=https://ip-of-the-master-node:6443 --certificate-authority="$(pwd)/k3s_ca"
or 
kubectl config set-cluster myk3scluster --server=https://ip-of-the-master-node:6443 --insecure-skip-tls-verify=true
kubectl config set-credentials service-user --token="$(cat ./k3s_token)"
kubectl config set-context k3oscluster --cluster=myk3scluster --user=service-user
kubectl config use-context k3oscluster
kubectl config view
```
- Start a proxy to the remote cluster
```
$ kubectl proxy --port=8002
```
- Open the Api in your local browser at ```http://localhost:8002/api/```.
- Install the latest version of the kubernete dashboard using helm (Download helm client at https://github.com/helm/helm/releases). Note: With Helm 3 you now have to use the form ```helm [command] [name] [chart]```
```
$ helm --kube-context k3oscluster repo add stable https://kubernetes-charts.storage.googleapis.com/
$ helm --kube-context k3oscluster list
$ helm --kube-context k3oscluster repo update
$ helm --kube-context k3oscluster install dashboard-demo stable/kubernetes-dashboard
NAME: dashboard-demo
LAST DEPLOYED: Mon Feb 10 17:35:07 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
*********************************************************************************
*** PLEASE BE PATIENT: kubernetes-dashboard may take a few minutes to install ***
*********************************************************************************

Get the Kubernetes Dashboard URL by running:
  export POD_NAME=$(kubectl get pods -n default -l "app=kubernetes-dashboard,release=dashboard-demo" -o jsonpath="{.items[0].metadata.name}")
  echo https://127.0.0.1:8443/
  kubectl -n default port-forward $POD_NAME 8443:8443
$ helm upgrade dashboard-demo stable/kubernetes-dashboard --set fullnameOverride="dashboard"
$ kubectl get services
```
- Check the dashboard at ```http://localhost:8002/api/v1/namespaces/default/services/https:dashboard:https/proxy/```
- Delete the dashboard afterwards ```helm delete dashboard-demo```

Getting the [K8s dashboard](https://www.thebookofjoel.com/cheap-production-k3s-with-dashboard-ui) running with it's [own namespace](https://akomljen.com/installing-kubernetes-dashboard-per-namespace/)

# Preconfigure k3OS config.yaml
The ```/k3os/system/config.yaml``` file is reserved for the system installation and should not be modified on a running system.
At runtime, the config can be changed by creating/modifying the ```/var/lib/rancher/k3os/config.yaml``` and ```/var/lib/rancher/k3os/config.d/*```

## Configure k3s (Kubernetes) within k3OS
All Kubernetes configuration is done by configuring k3s. This is primarily done through environment and k3s_args keys in config.yaml.
Default Environment variable can be added by modifying [environment variable](overlay/etc/environment).
The write_files key can be used to populate the /var/lib/rancher/k3s/server/manifests folder with apps you'd like to deploy on boot.
Any file found in [k3s /var/lib/rancher/k3s/server/manifests](overlay/share/rancher/k3s/server/manifests) will automatically be deployed to Kubernetes in a manner similar to kubectl apply.
It is also possible to deploy Helm charts. K3s supports a CRD controller for installing charts. 
See [k3s yaml manifests](https://github.com/rancher/k3s/tree/master/manifests) as examples.

see [k3s advanced options](https://rancher.com/docs/k3s/latest/en/advanced/) for configuring Helm, MetalLB, OpenEBS.

## SSH Public-Key
- Generate a public key fingerprint for you (SSH client on Windows: https://www.ssh.com/ssh/putty/windows/puttygen#running-puttygen)
- Add your public-key to [K3OS configuration file baked into the iso](images/07-iso/config.yaml)
```
  ssh_authorized_keys:
    - ssh-rsa YOURSSHPUBLICKEY rancher@myhostname
  hostname: myhostname
``` 

# Deploy in Windows 10 Hyper-V for dry-run tests
Create a V-2 virtual machine loading your iso file
Take care to:
- Don't use secure-boot.
- Use Default-Network

### Setup SSH access

### Access from Host-OS:
- In K3os-VM, run ```$ ifconfig```, and read under eth0 the ip-address
- In K3os-VM, run ```$ hostname
k3os-1962```
- Generate a public key fingerprint for your client (https://www.ssh.com/ssh/putty/windows/puttygen#running-puttygen)
  Output: 
  
  sudo vi /var/lib/rancher/k3os/config.yaml
  
  Add following ```
  ssh_authorized_keys:
  - ssh-rsa SHA256:d3Ykm9jBz1n/socoIZeFeL7c1PpnlOG7W7aR0CxagC8 rancher@k3os-1962
  ``` 

# Fork Maintainance
## Configure your fork of k3OS
1. Configure your remote fork
```
$ git remote -v
origin  https://github.com/ccuz/k3os.git (fetch)
origin  https://github.com/ccuz/k3os.git (push)
```
1. Add k3OS as upstream project
```
$ git remote add upstream https://github.com/rancher/k3os.git
```
1. Verify
```
$ git remote -v
origin  https://github.com/ccuz/k3os.git (fetch)
origin  https://github.com/ccuz/k3os.git (push)
upstream        https://github.com/rancher/k3os.git (fetch)
upstream        https://github.com/rancher/k3os.git (push)
```

## Merge upstream back into your fork of k3OS
1. Fetch the upstream changes ```$ git fetch upstream```
1. Create a new local branch to merge the upstream changes into your fork ```$ git checkout -b merge-upstream-in-my-fork```
1. Merge the upstream changes into my branch ```$ git merge upstream/master```
1. Check build and make a PR to merge your 'merge-upstream-in-my-fork' into your origin/master