# -*- mode: ruby -*-
# vi: set ft=ruby :

# Vagrantfile API/syntax version. Don't touch unless you know what you're doing!
VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  config.ssh.insert_key = false
  config.vm.box = "centos6.5_64"

  config.vm.define :db1 do |instance|
    instance.vm.network :private_network, ip:"10.11.22.101"
    config.vm.provider :virtualbox do |vb|
      vb.customize ["modifyvm", :id, "--memory", "1024"]
    end
  end

  # 以降、必要に応じてコメントを外す

  config.vm.define :db2 do |instance|
    instance.vm.network :private_network, ip:"10.11.22.102"
    config.vm.provider :virtualbox do |vb|
      vb.customize ["modifyvm", :id, "--memory", "1024"]
    end
  end

end

