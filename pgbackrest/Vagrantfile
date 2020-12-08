# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure("2") do |config|
  config.vm.define "primary" do |primary|
    primary.vm.box = "ubuntu/bionic64"
    primary.vm.network "private_network", ip: "192.168.34.10"
    primary.vm.hostname = "primary"
  end 

  config.vm.define "repository" do |repository|
    repository.vm.box = "ubuntu/bionic64"
    repository.vm.network "private_network", ip: "192.168.34.30"
    repository.vm.hostname = "repository"
  end 

  config.vm.define "replica" do |replica|
    replica.vm.box = "ubuntu/bionic64"
    replica.vm.network "private_network", ip: "192.168.34.20"
    replica.vm.hostname = "replica"
  end 
end
