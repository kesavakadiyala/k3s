# K3s Cluster Creation

## Goal:
This section provides instructions to deploy K3s light weight Kubernetes cluster in servers.

## Architecture:

![Architecture](https://github-production-user-asset-6210df.s3.amazonaws.com/28837244/245425286-15c83af2-4692-4b40-80dd-b52099e44415.png)


The above figure shows the difference between K3s server and K3s agent nodes. For more information, see the architecture documentation(https://k3s-io.github.io/docs/architecture).

Reference: https://k3s.io/

Kubernetes architecture involves a Master node and Worker Nodes. Their functions are as follows:

- Master – controls the cluster, API calls, e.t.c.
- Workers – these handles the workloads, where the pods are deployed and applications ran. They can be added and removed from the cluster.

So to setup a k3s cluster you need at least two hosts, the master node and one workernode. In this post we shall be using one master node and two worker nodes.

We need to prepare our hosts to be able to run k3s in a cluster. Use the following steps to install k3s cluster on Ubuntu.


## Prerequisites:
- The exercise was performed on an ubuntu machines, so this documents is supported on ubuntu.
- Need to have at least 2 ubuntu instances for Master and worker.
- Once the servers are up and running, check if the ports 80, 443 are opened to public 0.0.0.0/0 and ports 6443, 8472 are opened to subnet cidr range under security group. 
- Note: Above mentioned ports are specific to this POC alone. For more details on the port requirements of K3s, please refer to this document - Port Requirements | Rancher .

## Process Overview
### Step 1: Update Ubuntu system
  - With your servers installed with Ubuntu, update and upgrade them:

  ```
  sudo apt update
  sudo apt -y upgrade && sudo systemctl reboot
  ```

### Step 2: Map the Hostnames on each node
  - Make sure you have the hostnames mapped on each node. This is by adding the Private IP and hostname of each node in the /etc/hosts file of each host.
    
    In our setup, it is as follows:

  ```
  $ sudo vim /etc/hosts
  172.16.10.3 master
  172.16.10.4 worker01
  172.16.10.10 worker02
  ```

### Step 3: Install Docker on Ubuntu
The next step is to install docker on on the hosts. As discussed before, Kubernetes is used to manage Docker containers on hybrid cloud infrastructure. Thus we need to have docker up and running on all the nodes before we can setup K3s.
Add Docker APT repository:

```
sudo apt update
sudo apt install apt-transport-https ca-certificates curl software-properties-common -y
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
```
Install Docker CE on Ubuntu:

```
sudo apt update
sudo apt install docker-ce -y
```
This has to be done on all the hosts including the master node. After successful installation, start and enable the service.

```
sudo systemctl start docker
sudo systemctl enable docker
```
You can also check if the service is started and is running:

```
systemctl status docker
```
![docker](https://github.com/kesavakadiyala/k3s/assets/28837244/4c5271ac-5105-4124-928e-25f4aa87d840)

Add your user to Docker group to avoid typing sudo every time you run docker commands.

```
sudo usermod -aG docker ${USER}
newgrp docker
```

### Step 4: Setup the Master k3s Node
In this step, we shall install and prepare the master node. This involves installing the k3s service and starting it.

```
curl -sfL https://get.k3s.io | sh -s - --docker
```
Run the command above to install k3s on the master node. The script installs k3s and starts it automatically.

```
[INFO]  Finding release for channel stable
[INFO]  Using v1.18.9+k3s1 as release
[INFO]  Downloading hash https://github.com/rancher/k3s/releases/download/v1.18.9+k3s1/sha256sum-amd64.txt
[INFO]  Downloading binary https://github.com/rancher/k3s/releases/download/v1.18.9+k3s1/k3s
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Skipping /usr/local/bin/ctr symlink to k3s, command exists in PATH at /usr/bin/ctr
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
Created symlink /etc/systemd/system/multi-user.target.wants/k3s.service → /etc/systemd/system/k3s.service.
[INFO]  systemd: Starting k3s
```
To check if the service installed successfully, you can use:

```
systemctl status k3s
```
![k3s](https://github.com/kesavakadiyala/k3s/assets/28837244/a2592fb5-37d8-44bb-9c9f-1a0699516baa)


You can check if the master node is working by :

```
sudo kubectl get nodes -o wide
```
The output should be something like this:

```
root@mastre:~# sudo kubectl get nodes -o wide
NAME     STATUS   ROLES    AGE    VERSION        INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
master   Ready    master   3m2s   v1.18.9+k3s1   172.16.10.1   <none>        Ubuntu 20.04.1 LTS   5.4.0-52-generic   docker://19.3.8
```

### Step 5: Allow ports on firewall
We need to allow ports that will will be used to communicate between the master and the worker nodes. The ports are 443 and 6443.

```
sudo ufw allow 6443/tcp
sudo ufw allow 443/tcp
```

You need to extract a token form the master that will be used to join the nodes to the master.
On the master node:

```
sudo cat /var/lib/rancher/k3s/server/node-token
```

You will then obtain a token that looks like:

```
K1078f2861628c95aa328595484e77f831adc3b58041e9ba9a8b2373926c8b034a3::server:417a7c6f46330b601954d0aaaa1d0f5b
```

### Step 6: Install k3s on worker nodes and connect them to the master
The next step is to install k3s on the worker nodes. Run the commands below to install k3s on worker nodes:

```
curl -sfL http://get.k3s.io | K3S_URL=https://<master_IP>:6443 K3S_TOKEN=<join_token> sh -s - --docker
```
Where master_IP is the IP of the master node and join_token is the token obtained from the master. e.g:

```
curl -sfL http://get.k3s.io | K3S_URL=https://172.16.10.3:6443 K3S_TOKEN=K1078f2861628c95aa328595484e77f831adc3b58041e9ba9a8b2373926c8b034a3::server:417a7c6f46330b601954d0aaaa1d0f5b sh -s - --docker
```
You can verify if the k3s-agent on the worker nodes is running by:

```
sudo systemctl status k3s-agent
```
![k3s-agent](https://github.com/kesavakadiyala/k3s/assets/28837244/a161e7a6-dce1-4b52-976b-61e5b04d3c8f)

### Step 7: Verify that nodes have successfully been added to the cluster
To verify that our nodes have successfully been added to the cluster, run :

```
sudo kubectl get nodes
```
Your output should look like:

```
root@master:~# kubectl get nodes -o wide
NAME       STATUS   ROLES    AGE   VERSION        INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION       CONTAINER-RUNTIME
master     Ready    master   14m   v1.18.9+k3s1   172.16.10.1   <none>        Ubuntu 20.04.1 LTS   5.4.0-52-generic     docker://19.3.8
worker02   Ready    <none>   90s   v1.18.9+k3s1   172.16.10.2   <none>        Ubuntu 20.04.1 LTS   5.4.0-52-generic     docker://19.3.8
worker01   Ready    <none>   41s   v1.18.9+k3s1   172.16.10.3   <none>        Ubuntu 18.04.5 LTS   4.15.0-122-generic   docker://19.3.6
```
This shows that we have successfully setup our k3s cluster ready to deploy applications to it.

Once k3s cluster installed successfully, Verify namespaces by running below command.

```
kubectl get ns
```


### Step 8: Deploy helm Addons to K3s
curl -sfSLO "https://get.helm.sh/helm-v3.10.1-linux-amd64.tar.gz"
tar -xvzf helm-v3.10.1-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm

```
curl -sfSLO "https://get.helm.sh/helm-v3.10.1-linux-amd64.tar.gz"
tar -xvzf helm-v3.10.1-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
```
Check helm version by running below command

```
helm version
```

### Step 9: Exporting kubeconfig and validating helm
Run below command for exporting kubeconfig and list installed helm charts.

```
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
helm ls
```
![helm-ls](https://github.com/kesavakadiyala/k3s/assets/28837244/36f1c83e-f077-48d1-bf44-37aa41924bbb)
