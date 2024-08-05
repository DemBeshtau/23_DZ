# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.ssh.insert_key = false
  config.vm.box = "centos/7"
  
  config.vm.provider "virtualbox" do |v|
	  v.memory = 256
    v.cpus = 1
  end 
 
  #config.vm.provision "ansible" do |ansible|   
    #ansible.playbook = "provisioning/playbook.yml"
    #ansible.inventory_path = "provisioning/hosts"
    #ansible.host_key_checking = "true"
    #ansible.limit = "all"
    #ansible.become = "true"
  #end

  config.vm.define "ns01" do |ns01|
    ns01.vm.network "private_network", ip: "192.168.56.10"
    ns01.vm.hostname = "ns01"
  end

  config.vm.define "ns02" do |ns02|
    ns02.vm.network "private_network", ip: "192.168.56.11"
    ns02.vm.hostname = "ns02"
  end

  config.vm.define "client1" do |client1|
    client1.vm.network "private_network", ip: "192.168.56.15"
    client1.vm.hostname = "client1"
  end

  config.vm.define "client2" do |client2|
    client2.vm.network "private_network", ip: "192.168.56.16"
    client2.vm.hostname = "client2"
  end

  config.vm.provision "shell", inline: <<-SHELL
    sudo sed -i 's/\PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
    sudo systemctl restart sshd
    sed -i 's/mirrorlist=/#mirrorlist=/g' /etc/yum.repos.d/CentOS-Base.repo
    sed -i 's/#baseurl/baseurl/g' /etc/yum.repos.d/CentOS-Base.repo
    sed -i 's/mirror.centos.org/vault.centos.org/g' /etc/yum.repos.d/CentOS-Base.repo
  SHELL

end

