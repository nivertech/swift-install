#---------------------------------------------------------------------------
# Copyright IBM Corp. 2015, 2015 All Rights Reserved
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# Limitations under the License.
#---------------------------------------------------------------------------
# Written By George Goldberg (georgeg@il.ibm.com)
# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'json'
file = File.read('inventory/vagrant/swift_config.json')
$config_hash = JSON.parse(file)
$config_hash = $config_hash["vagrant"]
# a list of machine_type's
# each machine is of the for #{machine_type}#{machine_id}
# where machine_id is in (1..machines[machine_type][num])
machine_list = $config_hash["machines"].keys

ip_suffix = 2
$iptables = Hash.new
machines = $config_hash["machines"]

# be lazy , calculate an ip for each machine 
# beforehand , the final ip will be
# #{ip_prefix}.#{iptables[machine_type][machine_id]}
machine_list.each do |m|
  num = machines[m]["num"]
  if num > 0
    $iptables[m] = Hash.new
    (0..(num - 1)).each do |pid|
       $iptables[m][pid] = ip_suffix
       ip_suffix += 1
    end
  end
end

$machine_num = ip_suffix - 2
$current_machine = 1

def get_machine_box(machinetype)
     machine = $config_hash["machines"][machinetype]
     boxes = $config_hash["boxes"]
     
     if machine.has_key?("box")
        return boxes[machine["box"]] 
     else
        return boxes[$config_hash["box"]]
     end

end

def setup_storage_devices(server, machinetype, pid)
  machine = $config_hash["machines"][machinetype]
  disksize = 4
  if machine.has_key?("disk_size")
     disksize = machine["disk_size"]
  end

  box = get_machine_box(machinetype)
  disk_controller = box["disk_controller"] 

  server.vm.provider "virtualbox" do | v |
    
    (1..machine["disk"]).each do |did|
      file_to_disk = "./disk_#{machinetype}_#{pid}_storage_#{did}.vdi"
      unless File.exist?(file_to_disk)
        v.customize ['createhd', '--filename', file_to_disk, '--size', disksize * 1024]
      end
      v.customize ['storageattach', :id, '--storagectl', disk_controller, '--port', did, '--device', 0, '--type', 'hdd', '--medium', file_to_disk]
    end
  end
end


def ansible_provision(server)
  server.vm.provision :ansible do |ansible|
    ansible.playbook = "main-vagrant.yml"
    ansible.inventory_path = "inventory/vagrant/swift_dynamic_inventory.py"
    ansible.sudo = true
    #ansible.verbose = 'vvvv' 
    ansible.limit = 'all'
  end
end

def get_ip(hostname, pid)
  ip_prefix = $config_hash["ip_prefix"]
  ip_suffix = $iptables[hostname][pid]

  return "#{ip_prefix}.#{ip_suffix}"
end

# setup cpu and memory
def setup_basic_parameters(server , machinetype , pid)
  machine = $config_hash["machines"][machinetype]
  
  # setup cpu and memory for the machine
  server.vm.provider "virtualbox" do |v|
    if machine.has_key?("cpus")
       v.cpus = machine["cpus"]
    else   
       v.cpus = $config_hash["cpus"]
    end

    if machine.has_key?("memory")
       v.memory = machine["memory"]
    else   
       v.memory = $config_hash["memory"]
    end   
  end
end

def setup_box(server,machinetype,pid)
  box = get_machine_box(machinetype)
  server.vm.box = box["box"]
  if box.has_key?("box_url")
     server.vm.box_url = box["box_url"]
  end
end


def setup_machine(server,machinetype,pid)
  machine = $config_hash["machines"][machinetype]
  # each setup parameter can be local to a machine_type
  # or global
  
  # setup cpu and memory
  setup_basic_parameters(server,machinetype,pid)
  
  # setup vagrant box and its url
  setup_box(server,machinetype,pid)

  if machine.has_key?("disk")
    setup_storage_devices(server, machinetype, pid)
  end
  
  # setup ip address
  ip_address = get_ip(machinetype, pid)
  server.vm.network "private_network",ip: ip_address
  server.vm.hostname = "#{machinetype}#{pid}"
  
  # use default key , one for all , you dont really need
  # security for testing purposes , don't you ?
  server.ssh.insert_key = false

  # run ansible provision only after the last machine is
  # set up , to set up to provision the whole cluster at once
  if $current_machine == $machine_num
     ansible_provision(server)
  end   
  $current_machine += 1
end

Vagrant.configure(2) do |config|
  # setup dns virtual box to intercept dns requests
  # and convey it to host dns
  config.vm.provider :virtualbox do |vb|
    vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
    vb.customize ["modifyvm", :id, "--natdnsproxy1", "on"]
  end
 
  $config_hash["machines"].keys.each do |m|  
    machine = $config_hash["machines"][m]
    (0..(machine["num"] - 1)).each do |pid|
      config.vm.define "#{m}#{pid}" do |server|
        setup_machine(server,m,pid)
      end
    end
  end

end