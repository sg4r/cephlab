# -*- mode: ruby -*-
# vi: set ft=ruby :

ENV['VAGRANT_DEFAULT_PROVIDER'] = 'libvirt'


Vagrant.configure("2") do |config|

  config.vm.define "cephclt" do |config|
  config.vm.hostname = "cephclt"
  config.vm.box = "eurolinux-vagrant/rocky-8"
  config.vm.box_check_update = false
  config.ssh.forward_agent = true
  config.ssh.forward_x11 = true
  config.vm.network :private_network, ip: "192.168.111.10"
  config.vm.provision "shell", path: "configure.sh"
  config.vm.provider :libvirt do |v|
    v.memory = 1024
    end
  end

(1..4).each do |i|
  config.vm.define "cn#{i}" do |config|
  config.vm.hostname = "cn#{i}"
  config.vm.box = "eurolinux-vagrant/rocky-8"
  config.vm.box_check_update = false
  config.vm.network :private_network, ip: "192.168.111.#{i+10}"
  config.vm.provision "shell", path: "configure.sh"
  config.vm.provider :libvirt do |v|
    v.memory = 1024
    v.storage :file, :size => '40G'
    v.storage :file, :size => '50G' 
    end
  end
end

(1..2).each do |i|
  config.vm.define "cnrgw#{i}" do |config|
  config.vm.hostname = "cnrgw#{i}"
  config.vm.box = "eurolinux-vagrant/rocky-8"
  config.vm.box_check_update = false
  config.vm.network :private_network, ip: "192.168.111.#{i+15}"
  config.vm.provision "shell", path: "configure.sh"
  config.vm.provider :libvirt do |v|
    v.memory = 1024
    end
  end
end

end
