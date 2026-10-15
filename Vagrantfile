Vagrant.configure("2") do |config|

  # Ubuntu Server 24.04 LTS base box
  config.vm.box = "bento/ubuntu-24.04"
  config.vm.box_architecture = "amd64"

  # Common VM configuration
  config.vm.boot_timeout = 600

  # Controller node
  config.vm.define "controller01" do |node|
    node.vm.hostname = "controller01"
    node.vm.network "private_network",
      ip: "10.0.1.10"
    node.vm.network "private_network",
      ip: "192.168.58.10",
      auto_config: false,
      virtualbox__hostonly: "vboxnet2"

    node.vm.provider "virtualbox" do |vb|
      vb.name = "kolla-controller01"
      vb.memory = 7168
      vb.cpus = 2
    end

    node.disksize.size = "50GB"
  end

  # Compute node
  config.vm.define "compute01" do |node|
   node.vm.hostname = "compute01"
   node.vm.network "private_network", ip: "10.0.1.11"

   node.vm.provider "virtualbox" do |vb|
     vb.name = "kolla-compute01"
     vb.memory = 5120
     vb.cpus = 2

     # Enable nested virtualization for OpenStack Nova/KVM
     vb.customize ["modifyvm", :id, "--nested-hw-virt", "on"]
   end

   node.disksize.size = "60GB"
  end

  # Storage node
  config.vm.define "storage01" do |node|
    node.vm.hostname = "storage01"
    node.vm.network "private_network",
      ip: "10.0.1.12"

    node.vm.provider "virtualbox" do |vb|
      vb.name = "kolla-storage01"
      vb.memory = 2048
      vb.cpus = 1
    end

    node.disksize.size = "30GB"
  end

end
