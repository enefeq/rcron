# -*- mode: ruby -*-
# # vi: set ft=ruby :

INSTANCES = 3

Vagrant.configure("2") do |config|
  config.vm.box       = "ubuntu/trusty64"

  (1..INSTANCES).each do |i|
    config.vm.define vm_name = "rcl#{i}" do |rcl|
      rcl.vm.hostname = "rcron-cluster-#{i}"
      ip = "50.50.50.#{i+100}"
      rcl.vm.network :private_network, ip: ip

      rcl.vm.provision "ansible" do |ansible|
        ansible.playbook       = "test/playbook.yml"
        ansible.inventory_path = "test/local"
        ansible.limit          = ip
        #ansible.verbose        = "vvvv"
      end
    end
  end
end
