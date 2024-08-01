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
      sudo apt-get install -y apt-transport-https curl
      echo "deb [signed-by=/user/share/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
      curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /user/share/keyrings/kubernetes-apt-keyring.gpg
      sudo apt-get update
      sudo apt-get install -y kubelet kubeadm kubectl
      sudo apt-mark hold kubelet kubeadm kubectl
    SHELL
  end