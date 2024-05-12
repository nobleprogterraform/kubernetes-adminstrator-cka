# cka-kubernetes-administrator

Create 3 nodes cluster:
    vagrant up
 check ssh
    vagrant ssh controlplane
    vagrant ssh node01
    vagrant ssh node02

Cluster Installation: (Do below on all 3 nodes)
    1. Installation of kubeadm (https://v1-29.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
    1.1 Installing container runtime containderd
        https://v1-29.docs.kubernetes.io/docs/setup/production-environment/container-runtimes/
----Run below on all nodes for Forwarding IPv4 ------------------
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
-----------------------------
    Installing containerd https://docs.docker.com/engine/install/ubuntu/
    Set up Docker's apt repository.
    -----Run below on all 3 nodes-------------
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
-------------------------------------
    Install the Docker packages:
        sudo apt-get install containerd.io
        systemctl status containerd (to verify containerd is active)
    Configuring the systemd cgroup driver
        ps -p 1 (to check systemd cgroup)
        sudo vi /etc/containerd/config.toml (dG to delete all existing lines, paste below lines,save)
---------------------------------------------------
version = 2
[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
   [plugins."io.containerd.grpc.v1.cri".containerd]
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          runtime_type = "io.containerd.runc.v2"
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = true
----------------------------------------
        sudo systemctl restart containerd

    1.2 Installing kubeadm, kubelet and kubectl
        1.2.1 Update the apt package index and install packages needed to use the Kubernetes apt repository:
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
        1.2.2 Download the public signing key for the Kubernetes package repositories
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
        1.2.3 Add the appropriate Kubernetes apt repository
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
        1.2.4 Update the apt package index
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

    1.3 Creating a cluster with kubeadm (https://v1-29.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
    Run below only on controlplane node
    sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=10.0.32.254
    run the commands (output of previous commands)to copy kube-config file to home directory
    kubeadm token create --print-join-command (in case you cleard the terminal, will need after pod networking)

    1.4 Installing Addons(https://v1-29.docs.kubernetes.io/docs/concepts/cluster-administration/addons/#networking-and-network-policy)
     Run below on master node only:
     wget https://reweave.azurewebsites.net/k8s/v1.29/net.yaml
     vi net.yaml
     add below env variabels under container named weave
     name: IPALLOC_RANGE
     value: 10.244.0.0/16

     1.5 Join the worker nodes in the cluster
    kubeadm token create --print-join-command (to print the join command to be run on workder nodes)
    sudo replace_with_join_commands (only run on all worker nodes)
    verify by running a container: kubectl run nginx --image=nginx
