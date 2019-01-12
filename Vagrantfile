# -*- mode: ruby -*-
# vi: set ft=ruby :

VAGRANTFILE_API_VERSION = "2"

ENV["ANSIBLE_NOCOWS"] = "True"

# Database server
Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  config.vm.box = "centos/7"
  config.vm.box_version = "1804.02"
  config.vm.box_check_update = false
  config.vm.synced_folder ".", "/vagrant", disabled: true

  config.vm.provider :virtualbox do |vb|
    vb.customize ["modifyvm", :id, "--ioapic", "on"]
    vb.customize ["modifyvm", :id, "--memory", "256"]
    vb.customize ["modifyvm", :id, "--cpus", "1"]
  end

  config.vm.define :h0 do |mybox|
    mybox.vm.hostname = "h0"
    mybox.vm.network :private_network,
                     ip: "192.168.60.10",
                     netmask: "255.255.255.0",
                     auto_config: false
    mybox.vm.provision "h0-nic", type: "shell" do |p|
      p.inline = <<-SCRIPT
        yum install -y tcpdump
        nmcli dev set eth1 managed no
        ip link set eth1 down                                # ifconfig eth1 down
        ip address add 192.168.60.10/24 broadcast + dev eth1 # ifconfig eth1 192.168.60.10 netmask 255.255.255.0
        ip link set eth1 up                                  # ifconfig eth1 up
      SCRIPT
    end
  end

  config.vm.define :h1 do |mybox|
    mybox.vm.hostname = "h1"
    mybox.vm.network :private_network,
                     ip: "192.168.61.11",   # n.b., this is just to force virtualbox to create a separate
                                            #       network for this NIC; this is NOT the IP address we
                                            #       intend to run this NIC on; cf. `ip address add` below.
                     netmask: "255.255.255.0",
                     auto_config: false
    mybox.vm.provision "h1-nic", type: "shell" do |p|
      p.inline = <<-SCRIPT
        yum install -y tcpdump
        nmcli dev set eth1 managed no
        ip link set eth1 down
        ip address add 192.168.60.11/24 broadcast + dev eth1
        ip link set eth1 up
      SCRIPT
    end
  end

  config.vm.define :b0 do |mybox|
    mybox.vm.hostname = "b0"
    mybox.vm.network :private_network, ip: "192.168.60.9", netmask: "255.255.255.0", auto_config: false
    mybox.vm.network :private_network, ip: "192.168.61.9", netmask: "255.255.255.0", auto_config: false
    mybox.vm.provider "virtualbox" do |vb|
      vb.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
      vb.customize ["modifyvm", :id, "--nicpromisc3", "allow-all"]
    end
    mybox.vm.provision "b0-bridge", type: "shell" do |p|
      p.inline = <<-SCRIPT
        yum install -y bridge-utils tcpdump
        nmcli dev set eth1 managed no
        ip link set eth1 down
        ip address flush dev eth1
        ip link set eth1 promisc on
        ip link set eth1 up
        nmcli dev set eth2 managed no
        ip link set eth2 down
        ip address flush dev eth2
        ip link set eth2 promisc on
        ip link set eth2 up
        brctl addbr bridge0
        nmcli dev set bridge0 managed no
        brctl addif bridge0 eth1
        brctl addif bridge0 eth2
        ip link set bridge0 up
      SCRIPT
    end
  end

end
