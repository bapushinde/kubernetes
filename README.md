## Setup Multinode Kubernetes cluster quickly on Ubuntu VMs

## Prerequisite:
  a. 3 VMs having specification of (**2 vCPUs, 2 GB memory**, 10GB storage)
  
  b. Open ports **22, 80, 443, 6443** within VPC/Compute Network


### Simplified steps to setup cluster (1 Master & 2 Worker Nodes). 

*(P.S. These step are Not limited to 2 worker nodes only, If you have `N` number of worker nodes, just repeat worker nodes steps on all the worker nodes)*

#### Setup Docker Runtime on each VM

  1. SSH into all 3 VMs & switch to root user using `sudo -i`

  2. Open URL -> https://kubernetes.io/docs/setup/production-environment/container-runtimes/#docker in browser (or another tab window) 

  3. Select **Ubuntu 16.04+** Tab to executes commands on all **3 VMs to setup Docker runtime**.
         
        *Tip: Last command executed through this link should be* 
         
        ```console 
          sudo systemctl enable docker
        ``` 
  
  4. Run `docker ps` on each VM and you should see output 
  ```console
root@terraform-k8s-worker-1:~# docker ps 
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
root@terraform-k8s-worker-1:~# 
```

  5. Execute below **Prerequisite** on all 3 VMs (*Don't miss this step during docker runtime Setup*): 
  ```console
      sudo modprobe overlay
      sudo modprobe br_netfilter
      
      # Set up required sysctl params, these persist across reboots.
      
      cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
      net.bridge.bridge-nf-call-iptables  = 1
      net.ipv4.ip_forward                 = 1
      net.bridge.bridge-nf-call-ip6tables = 1
      EOF
      
      sudo sysctl --system
 ```

#### Install Kubernetes Orachestration tool "kubeadm" on all Nodes.

   1. Open kubernetes documentation link -> https://kubernetes.io/docs/home/ and enter text search `install kubeadm` in search box on left hand side
   
   2. Open this link from *Search Result* -> https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
      
      On this page Jump on to section **Installing kubeadm, kubelet and kubectl** 
      
      Select Tab **Ubuntu, Debian Or HypriotOS** and execute following set of commands from this section **on all 3 VMs**:
      
      ```console
      sudo apt-get update && sudo apt-get install -y apt-transport-https curl
      
      curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
      
      cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
      deb https://apt.kubernetes.io/ kubernetes-xenial main
      EOF
      
      sudo apt-get update
      
      sudo apt-get install -y kubelet kubeadm kubectl
      
      sudo apt-mark hold kubelet kubeadm kubectl
      ```
   
   3. Restarting the kubelet is required:
   
   ```console
   systemctl daemon-reload
   systemctl restart kubelet
   ```
#### Setup Master Node in the Cluster
*Important!:* To setup master node in the cluster please Switch to Master Node VM terminal

1. Open link for refrerence to run **`kubeadm init`** on **Master node VM**  --> https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/ 

On this page Jump on to section **More information** 

Execute kubeadm initilisation command

```console
kubeadm init

# You will see output in console like.. 

[init] Using Kubernetes version: vX.Y.Z
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Activating the kubelet service
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "etcd/ca" certificate and key
.
.
.
.

[bootstraptoken] creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a Pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  /docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>

```
2. Copy the following `kubeadm init` command output, into handy text editor like `notepad/notepad++/sublimetext` etc
```console
To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a Pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  /docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```
3. To make kubectl work for your **non-root user** from **Master Node**, run these commands 
   
   1. Exit from root-user for example
  ```console
  root@terraform-k8s-master-0:~# exit
  logout
  bapus_pro_2020@terraform-k8s-master-0:~$
  ```
   2. Run as a normal user:
  ```console
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
  ```
   3. Now run `kubectl get nodes` command, there should be only one node *(i.e. Master node in Not Ready Status)*
  ```console
  bapus_pro_2020@terraform-k8s-master-0:~$ kubectl get nodes
  NAME                         STATUS   ROLES    AGE   VERSION
  terraform-k8s-master-0   Not Ready    master   40s   v1.19.2
  bapus_pro_2020@terraform-k8s-master-0:~$ 
  ```
4. Now to apply Network policy to establish network communication within cluster between master node & worker node. And get *Master node* in *ready* state. Here we will setup Calico network policy
    
    1. Open link to setup calico network policy on **Master node** --> https://docs.projectcalico.org/getting-started/kubernetes/self-managed-onprem/onpremises
      
      **Tip:** To open calico network policy documentation through Google search, *Google search "calico.yaml kubernetes"* and open first link **"Install Calico networking and network policy for on-premises ..."**
    
    2. Now Run `kubectl apply -f <filename_or_url>` command on Master node
      ```console
      curl https://docs.projectcalico.org/manifests/calico.yaml -O
      kubectl apply -f calico.yaml
      ```
      OR 
      ```console 
      kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
      ```
    3. Check Network policy apply run result. Execute `kubectl get nodes -w` & wait for command to output ready status
    ```console 
    kubectl get nodes -w
    NAME                         STATUS   ROLES    AGE   VERSION
    terraform-k8s-master-0   Not Ready    master   2m9s   v1.19.2
    terraform-k8s-master-0   Not Ready    master   2m10s   v1.19.2
    terraform-k8s-master-0   Not Ready    master   2m11s   v1.19.2
    terraform-k8s-master-0   Ready        master   2m12s   v1.19.2
    terraform-k8s-master-0   Ready        master   2m13s   v1.19.2
    terraform-k8s-master-0   Ready        master   2m14s   v1.19.2
    .
    .
    #Now hit Control+c 
    ```
    4. Now run `kubectl get nodes` command, 
    ```console
    bapus_pro_2020@terraform-k8s-master-0:~$ kubectl get nodes
    NAME                         STATUS   ROLES    AGE   VERSION
    terraform-k8s-master-0   Ready    master   2m40s   v1.19.2
    bapus_pro_2020@terraform-k8s-master-0:~$ 
    ```
#### Now Join each worker Nodes to Master node in the cluster by running `kubeadm join` command from Worker node VM 

*Important!:* To join worker node in the cluster please Switch to Worker Node VM terminal (Do this for each worker node)

1. Copy `kubeadm join xxxxxx ` command from text editor previously copied as `kubeadm init` command output & run on each **Worker Node**
i.e.  
```console 
kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```
Example:

```console
root@terraform-k8s-worker-1:~# kubeadm join 10.142.0.4:6443 --token fia5f8.2x5ud7uhwgxlcfb2 \
> --discovery-token-ca-cert-hash sha256:42b95610799f923b5ffbc770dd5951a7870aa53efb1ba28938bec69e208e30fd 
[preflight] Running pre-flight checks

[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.

root@terraform-k8s-worker-1:~# 
```
**Repeat this step for each Worker Node**

2. Now Switch to **Master Node** VM terminal and run `kubectl get nodes` command 
```console 
bapus_pro_2020@terraform-k8s-master-0:~$ kubectl get nodes
NAME                         STATUS   ROLES    AGE   VERSION
terraform-k8s-master-0   Ready    master   3m5s   v1.19.2
terraform-k8s-worker-0   Ready    <none>   3s   v1.19.2
terraform-k8s-worker-1   Ready    <none>   4s   v1.19.2
bapus_pro_2020@terraform-k8s-master-0:~$ 
```

## Congratulations, Here you've Successfully setup Kubernetes Cluster (1 Master Node and 2 Worker Nodes)!!
