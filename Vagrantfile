# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.box = "centos/7"

  
  config.vm.provider "virtualbox" do |v|
    v.memory = 1024
    v.cpus = 2
  end
  
  config.vm.define "rpmlab" do |nfss|
    nfss.vm.network "private_network", ip: "192.168.50.10", virtualbox__intnet: "net1"
    nfss.vm.hostname = "rpmlab"
    nfss.vm.provision "shell", path: "rpm.sh"
  end

end
