# -*- mode: ruby -*-
# vi: set ft=ruby :

ENV['VAGRANT_DEFAULT_PROVIDER'] = 'libvirt'
#ENV['VAGRANT_DEFAULT_PROVIDER'] = 'virtualbox'


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
  config.vm.provider :virtualbox do |v|
    v.memory = 1024
    v.linked_clone = true
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
    v.memory = 1600
    v.storage :file, :size => '40G'
    v.storage :file, :size => '50G' 
    end
  config.vm.provider :virtualbox do |v|
    v.memory = 1600
    v.linked_clone = true
    unless File.exist?("disk-#{i}-0.vdi")
    # Adding OSD Controller;
    v.customize ['storagectl', :id,
                   '--name', 'OSD Controller',
                   '--add', 'sata']
    end
    (0..1).each do |d|
         v.customize ['createhd',
                        '--filename', "disk-#{i}-#{d}",
                        '--size', '40000'] unless File.exist?("disk-#{i}-#{d}.vdi")
          v.customize ['storageattach', :id,
                        '--storagectl', 'OSD Controller',
                        '--port', 3 + d,
                        '--device', 0,
                        '--type', 'hdd',
                        '--medium', "disk-#{i}-#{d}.vdi"]
    end
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
    v.memory = 1600
    end
  config.vm.provider :virtualbox do |v|
    v.memory = 1600
    v.linked_clone = true
    end
  end
end

end
