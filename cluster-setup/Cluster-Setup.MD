## Steps to setup kubernetes cluster using shell scripts.
1. SSH into **Master Node** VM 
2. Download cluster-setup/install_master.sh
3. Run install Master node shell script 
```console
./install_master.sh
```
4. Wait for script to be completed
5. SSH into **Worker Node** VM
6. Download cluster-setup/install_worker.sh
7. Run install worker node shell script
  ```console 
  ./install_worker.sh
  ```
8. Wait for script to be completed
9. Join worker node to Master node using join command from Master node 
Example: 
```console
kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

