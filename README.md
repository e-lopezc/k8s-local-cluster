#  Installing a Kubernetes Cluster with Vagrant and VirtualBox on Ubuntu 22.04

As part of my learning just documented the process of setting up a Kubernetes cluster with one master node and two worker nodes using Vagrant and VirtualBox on an Ubuntu 22.04 host machine.

## Prerequisites

1. Ubuntu 22.04 LTS installed on your local machine
2. VirtualBox installed
3. Vagrant installed

If you haven't installed VirtualBox and Vagrant, you can do so by running:

```bash
sudo apt update
sudo apt install virtualbox vagrant -y
```

## Step 1: Create a new directory for your Kubernetes cluster

```bash
mkdir ~/kube-cluster
cd ~/kube-cluster
```

## Step 2: Create a Vagrantfile

Create a file named `Vagrantfile` in the `~/kube-cluster` directory with the following content:

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/focal64"

  config.vm.provider "virtualbox" do |vb|
    vb.memory = 2048
    vb.cpus = 2
  end

  config.vm.define "master" do |master|
    master.vm.hostname = "master"
    master.vm.network "private_network", ip: "172.16.0.10"
  end

  (1..2).each do |i|
    config.vm.define "worker-#{i}" do |worker|
      worker.vm.hostname = "worker-#{i}"
      worker.vm.network "private_network", ip: "172.16.0.#{i + 10}"
    end
  end

  config.vm.provision "shell", inline: <<-SHELL
    sudo apt-get update
    sudo apt-get install -y docker.io
    sudo systemctl enable docker
    sudo systemctl start docker
    sudo apt-get install -y apt-transport-https curl gpg
    echo "deb [signed-by=/user/share/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /user/share/keyrings/kubernetes-apt-keyring.gpg
    sudo apt-get update
    sudo apt-get install -y kubelet kubeadm kubectl
    sudo apt-mark hold kubelet kubeadm kubectl
  SHELL
end
```

## Step 3: Start the VMs

Run the following command to start the VMs:

```bash
vagrant up
```

This process may take some time as it downloads the Ubuntu image and installs the necessary packages.

## Step 4: Initialize the Kubernetes cluster on the master node

SSH into the master node:

```bash
vagrant ssh master
```

Initialize the Kubernetes cluster:

```bash
sudo kubeadm init --apiserver-advertise-address=172.16.0.10 --pod-network-cidr=10.244.0.0/16
```


After the initialization is complete, you'll see instructions for setting up kubectl. Follow those instructions:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Install a pod network add-on (we'll use Flannel in this example):

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

You'll see a join command for worker nodes. Copy this command as you'll need it in the next step.

Exit the master node:

```bash
exit
```

## Step 5: Join worker nodes to the cluster

For each worker node, SSH into the node and run the join command you copied earlier. Replace `<join-command>` with the actual command:

```bash
vagrant ssh worker-1
sudo <join-command>
exit

vagrant ssh worker-2
sudo <join-command>
exit
```

join-command example:
```bash
kubeadm join 172.16.0.10:6443 --token xxxxxxxxxxxxxxx --discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

## Step 6: Verify the cluster

SSH back into the master node:

```bash
vagrant ssh master
```

Check the status of the nodes:

```bash
kubectl get nodes
```

You should see all three nodes (one master and two workers) in the "Ready" state.

## Step 7: Test the cluster

To test your cluster, you can deploy a simple application:

```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
```

Get the NodePort:

```bash
kubectl get svc nginx
```

You can now access the nginx service from your host machine by visiting `http://172.16.0.11:<NodePort>` or `http://172.16.0.12:<NodePort>` in your web browser.

## Conclusion

You now have a functioning Kubernetes cluster with one master node and two worker nodes running on your local Ubuntu 22.04 machine using Vagrant and VirtualBox. You can start deploying your applications to this cluster for testing and development purposes.

Remember to shut down the VMs when you're done:

```bash
cd ~/kube-cluster
vagrant halt
```

To destroy the VMs completely:

```bash
vagrant destroy -f
```
