# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.box = "centos/7"

  # config.vm.provision "ansible" do |ansible|
  #   ansible.verbose = "vvv"
  #   ansible.playbook = "provisioning/playbook.yml"
  #   ansible.sudo = "true"
  # end

  config.vm.provider "virtualbox" do |v|
	  v.memory = 256
  end

  config.vm.define "ns01" do |ns01|
    ns01.vm.network "private_network", ip: "192.168.56.10"#, virtualbox__intnet: "mynet"
    ns01.vm.hostname = "ns01"
  end

  config.vm.define "ns02" do |ns02|
    ns02.vm.network "private_network", ip: "192.168.56.11"#, virtualbox__intnet: "mynet"
    ns02.vm.hostname = "ns02"
  end

  config.vm.define "client1" do |client1|
    client1.vm.network "private_network", ip: "192.168.56.15"#, virtualbox__intnet: "mynet"
    client1.vm.hostname = "client1"
  end

  config.vm.define "client2" do |client2|
    client2.vm.network "private_network", ip: "192.168.56.16"#, virtualbox__intnet: "mynet"
    client2.vm.hostname = "client2"
  end

  ssh_pub_key = File.readlines("../id_rsa.pub").first.strip
  config.vm.provision "shell", inline: <<-SHELL
    echo #{ssh_pub_key} >> ~vagrant/.ssh/authorized_keys
    mkdir -p ~root/.ssh
    echo #{ssh_pub_key} >> ~root/.ssh/authorized_keys
    sudo sed -i 's/\#PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
    systemctl restart sshd
    sed -i 's/mirrorlist=/#mirrorlist=/g' /etc/yum.repos.d/CentOS-Base.repo
    sed -i 's/#baseurl/baseurl/g' /etc/yum.repos.d/CentOS-Base.repo
    sed -i 's/mirror.centos.org/vault.centos.org/g' /etc/yum.repos.d/CentOS-Base.repo
  SHELL

end
