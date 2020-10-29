# Guide for metrics API server setup in kubernetes cluster build manually

### Step 1: ssh into Master node (Control pane)

### Step 2: git clone repo from [kubernetes-metrics-server](https://github.com/kodekloudhub/kubernetes-metrics-server.git)
Example : git clone <repo_url>

### Step 3: cd kubernetes-metrics-server

### Step 4: kubectl create -f .

### Step 5: kubectl get all --all-namespaces 

Verify that metric-server components **pod/service/deployment/replicaset** are in **running** status
