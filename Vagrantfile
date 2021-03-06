# -*- mode: ruby -*-
# vi: set ft=ruby :

# Uncomment the next line to force use of VirtualBox provider when Fusion provider is present
# ENV['VAGRANT_DEFAULT_PROVIDER'] = 'virtualbox'

# Nodes: logging 	192.168.100.201
#        controller-01 	192.168.100.198
#        controller-02 	192.168.100.199
#        controller-03 	192.168.100.200
#        compute-01 	192.168.100.202
#        compute-02 	192.168.100.203

# Interfaces
# eth0 - nat (used by VMware/VirtualBox)
# eth1 - br-mgmt (Container) 172.16.0.0/16
# eth2 - br-vlan (Neutron VLAN network) 0.0.0.0/0
# eth3 - host / API 192.168.100.0/24
# eth4 - br-vxlan (Neutron VXLAN Tunnel network) 172.29.240.0/22

nodes = {
    'logging'  => [1, 201],
    'controller' => [3, 198],
    'compute'  => [1, 202],
}

Vagrant.configure("2") do |config|
    
  # Virtualbox
  config.vm.box = "bunchc/trusty-x64"
  #config.vm.box = "ubuntu/trusty64"
  config.vm.synced_folder ".", "/vagrant", type: "nfs"

  # VMware Fusion / Workstation
  config.vm.provider :vmware_fusion or config.vm.provider :vmware_workstation do |vmware, override|
    override.vm.box = "bunchc/trusty-x64"
    override.vm.synced_folder ".", "/vagrant", type: "nfs"

    # Fusion Performance Hacks
    vmware.vmx["logging"] = "FALSE"
    vmware.vmx["MemTrimRate"] = "0"
    vmware.vmx["MemAllowAutoScaleDown"] = "FALSE"
    vmware.vmx["mainMem.backing"] = "swap"
    vmware.vmx["sched.mem.pshare.enable"] = "FALSE"
    vmware.vmx["snapshot.disabled"] = "TRUE"
    vmware.vmx["isolation.tools.unity.disable"] = "TRUE"
    vmware.vmx["unity.allowCompostingInGuest"] = "FALSE"
    vmware.vmx["unity.enableLaunchMenu"] = "FALSE"
    vmware.vmx["unity.showBadges"] = "FALSE"
    vmware.vmx["unity.showBorders"] = "FALSE"
    vmware.vmx["unity.wasCapable"] = "FALSE"
    vmware.vmx["vhv.enable"] = "TRUE"
  end

  #Default is 2200..something, but port 2200 is used by forescout NAC agent.
  config.vm.usable_port_range= 2800..2900 

  #if Vagrant.has_plugin?("vagrant-cachier")
  #  config.cache.scope = :box
  #  config.cache.enable :apt
  #  config.cache.synced_folder_opts = {
  #    type: :nfs,
  #    mount_options: ['rw', 'vers=3', 'tcp', 'nolock']
  #  }
  #else
  #  puts "[-] WARN: This would be much faster if you ran vagrant plugin install vagrant-cachier first"
  #end

  nodes.each do |prefix, (count, ip_start)|
    count.times do |i|
      if prefix == "compute" or prefix == "controller"
        hostname = "%s-%02d" % [prefix, (i+1)]
      else
        hostname = "%s" % [prefix, (i+1)]
      end

      config.vm.define "#{hostname}" do |box|
        box.vm.hostname = "#{hostname}.cook.book"
        box.vm.network :private_network, ip: "172.16.0.#{ip_start+i}", :netmask => "255.255.0.0"
        box.vm.network :private_network, ip: "10.10.0.#{ip_start+i}", :netmask => "255.255.255.0" 
      	box.vm.network :private_network, ip: "192.168.100.#{ip_start+i}", :netmask => "255.255.255.0" 
      	box.vm.network :private_network, ip: "172.29.240.#{ip_start+i}", :netmask => "255.255.252.0" 


	# Logging host is also the deployment server, so this will have the master SSH key which then gets copied
	if prefix == "logging"
	  box.vm.provision :shell, :path => "masterkey.sh"
	else
	  box.vm.provision :shell, :path => "clientkey.sh"
	end

        box.vm.provision :shell, :path => "hosts.sh"
        box.vm.provision :shell, :path => "install-prereqs.sh"
        box.vm.provision :shell, :path => "networking.sh"

	# Assumption is that compute-01 will run last, change to suit
	# This will shell into logging to kickstart the install from it
	if hostname == "compute-01"
	  box.vm.provision :shell, :path => "bootstrap-install.sh"
	end

        # box.vm.provision :shell, :path => "#{prefix}.sh"

        # If using Fusion or Workstation
        box.vm.provider :vmware_fusion or box.vm.provider :vmware_workstation do |v|
          v.vmx["memsize"] = 2048
          if prefix == "controller"
            v.vmx["memsize"] = 2048
            v.vmx["numvcpus"] = "1"
          end
          if prefix == "compute"
            v.vmx["memsize"] = 2048
            v.vmx["numvcpus"] = "1"
          end
        end

        # Otherwise using VirtualBox
        box.vm.provider :virtualbox do |vbox|
          # Defaults
          vbox.customize ["modifyvm", :id, "--memory", 1024]
          vbox.customize ["modifyvm", :id, "--cpus", 1]
          if prefix == "compute" or prefix == "controller"
            vbox.customize ["modifyvm", :id, "--memory", 3172]
            vbox.customize ["modifyvm", :id, "--cpus", 2]
          end
          vbox.customize ["modifyvm", :id, "--nicpromisc3", "allow-all"]
          vbox.customize ["modifyvm", :id, "--nicpromisc4", "allow-all"]
        end
      end
    end
  end
end
