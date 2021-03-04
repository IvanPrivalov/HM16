# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.box = "centos/7"
#  config.vbguest.auto_update = false

  config.vm.define "web" do |web|
    web.vm.network "private_network", ip: "192.168.10.10"
    web.vm.provider "virtualbox" do |v|
      v.memory = 512
      v.cpus = 1
    end
    web.vm.hostname = "web"
  end

  config.vm.define "log" do |log|
    log.vm.network "private_network", ip: "192.168.10.20"
    log.vm.provider "virtualbox" do |v|
      v.memory = 512
      v.cpus = 1
    end
    log.vm.hostname = "log"
  end

  config.vm.provision "RSYSLOG", type:'ansible' do |ansible|
    ansible.inventory_path = './inventories/all.yml'
    ansible.playbook = './logging.yml'
  end
end