# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
#
#svc_net_prefix = '192.168.202'

Vagrant.configure("2") do |config|
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://vagrantcloud.com/search.
  #
  #
  #config.vm.box = "rhcos/4.4.3"

  # Mac addresses calculated using
  # gen-virt-mac-from-ip () {
  #   printf '52:54:00:%02X:%02X:%02X\n' $(echo $1| cut -d'.' --output-delimiter=' ' -f2,3,4)
  # }



  config.vm.provider :libvirt do |libvirt|
    libvirt.cpus = 4
    libvirt.memory = 4096
    libvirt.nested = true
    libvirt.graphics_type = 'spice'
    libvirt.video_type = 'virtio'
  end

  config.vm.synced_folder ".",
    "/vagrant",
    type: "nfs",
    nfs_udp: false,
    disabled: true


  config.vm.define :lb, autostart: false do |node|
    node.vm.box = "centos/7"
    node.vm.network :private_network,
      :libvirt__network_name => 'okd-internal',
      :libvirt__dhcp_enabled => false,
      :ip => "192.168.100.2"
    node.vm.provider :libvirt do |libvirt|
      libvirt.cpus = 2
      libvirt.memory = 2048
    end
    node.vm.provision "ansible" do |ansible|
      ansible.playbook = "ansible/lb.yml"
    end
  end


  config.vm.define :bootstrap, autostart: false do |node|
    node.vm.network :private_network,
      :libvirt__network_name => 'okd-internal',
      :libvirt__dhcp_enabled => false,
      :mac => "52:54:00:A8:64:05"  # 192.168.100.5
    node.vm.provider :libvirt do |libvirt|
      libvirt.memory = 8192
      libvirt.boot 'hd'
      libvirt.boot 'network'
      libvirt.storage :file, :size => '20G', :type => 'qcow2'
      libvirt.mgmt_attach = false
    end
  end

  mac = [
    '52:54:00:A8:64:0A', # 192.168.100.10
    '52:54:00:A8:64:0B', # 192.168.100.11
    '52:54:00:A8:64:0C', # 192.168.100.12
  ]

  (0..2).each do |node_num|
    config.vm.define "cp#{node_num}", autostart: false do |node|
      node.vm.network :private_network,
        :libvirt__network_name => 'okd-internal',
        :libvirt__dhcp_enabled => false,
        :mac => mac[node_num]
      node.vm.provider :libvirt do |libvirt|
        libvirt.memory = 8192
        libvirt.boot 'hd'
        libvirt.boot 'network'
        libvirt.storage :file, :size => '40G', :type => 'qcow2'
        libvirt.mgmt_attach = false
      end
    end
  end

  (0..2).each do |node_num|
    config.vm.define "worker#{node_num}", autostart: false do |node|
      node.vm.network :private_network,
        :libvirt__network_name => 'okd-internal',
        :libvirt__dhcp_enabled => false
      node.vm.provider :libvirt do |libvirt|
        libvirt.memory = 8192
        libvirt.boot 'hd'
        libvirt.boot 'network'
        libvirt.storage :file, :size => '40G', :type => 'qcow2'
        libvirt.mgmt_attach = false
      end
    end
  end
end
