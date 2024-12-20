# -*- mode: ruby -*-
# vi:set ft=ruby sw=2 ts=2 sts=2:

# Set the build mode
BUILD_MODE = "BRIDGE"

# Define the number of worker nodes
NUM_WORKER_NODES = 2

# Static IP configuration
MASTER_IP = "192.168.1.100"
NODE_IPS = ["192.168.1.101", "192.168.1.102"]

# Host operating system detection
module OS
  def OS.windows?
    (/cygwin|mswin|mingw|bccwin|wince|emx/ =~ RUBY_PLATFORM) != nil
  end

  def OS.mac?
    (/darwin/ =~ RUBY_PLATFORM) != nil
  end

  def OS.unix?
    !OS.windows?
  end

  def OS.linux?
    OS.unix? and not OS.mac?
  end

  def OS.jruby?
    RUBY_ENGINE == "jruby"
  end
end

# Determine host adapter for bridging in BRIDGE mode
def get_bridge_adapter()
  if OS.windows?
    return %x{powershell -Command "Get-NetRoute -DestinationPrefix 0.0.0.0/0 | Get-NetAdapter | Select-Object -ExpandProperty InterfaceDescription"}.chomp
  elsif OS.linux?
    return %x{ip route | grep default | awk '{ print $5 }'}.chomp
  elsif OS.mac?
    return %x{mac/mac-bridge.sh}.chomp
  end
end

# Helper method to get the machine ID of a node.
def get_machine_id(vm_name)
  machine_id_filepath = ".vagrant/machines/#{vm_name}/virtualbox/id"
  if not File.exist? machine_id_filepath
    return nil
  else
    return File.read(machine_id_filepath)
  end
end

# Helper method to determine whether all nodes are up
def all_nodes_up()
  if get_machine_id("staticmaster").nil?
    return false
  end

  ["kubenode3", "kubenode4"].each do |node_name|
    if get_machine_id(node_name).nil?
      return false
    end
  end
  return true
end

# Sets up hosts file and DNS
def setup_dns(node)
  node.vm.provision "setup-hosts", :type => "shell", :path => "ubuntu/vagrant/setup-hosts.sh" do |s|
    s.args = [MASTER_IP, BUILD_MODE, NUM_WORKER_NODES, MASTER_IP] + NODE_IPS
  end
  node.vm.provision "setup-dns", type: "shell", :path => "ubuntu/update-dns.sh"
end

# Runs provisioning steps that are required by masters and workers
def provision_kubernetes_node(node)
  setup_dns node
  node.vm.provision "setup-ssh", :type => "shell", :path => "ubuntu/ssh.sh"
end

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/bionic64"
  config.vm.boot_timeout = 900
  config.vm.box_check_update = false

  # Provision master Node
  config.vm.define "staticmaster" do |node|
    node.vm.provider "virtualbox" do |vb|
      vb.name = "staticmaster"
      vb.memory = 2048
      vb.cpus = 2
    end
    node.vm.hostname = "staticmaster"
    if BUILD_MODE == "BRIDGE"
      node.vm.network :public_network, ip: MASTER_IP, bridge: get_bridge_adapter()
    else
      node.vm.network :private_network, ip: MASTER_IP
      node.vm.network "forwarded_port", guest: 22, host: "#{2710}"
    end
    provision_kubernetes_node node
    node.vm.provision "file", source: "./ubuntu/tmux.conf", destination: "$HOME/.tmux.conf"
    node.vm.provision "file", source: "./ubuntu/vimrc", destination: "$HOME/.vimrc"
  end

  # Provision Worker Nodes
  NODE_IPS.each_with_index do |node_ip, index|
    node_name = "kubenode#{index + 3}"
    config.vm.define node_name do |node|
      node.vm.provider "virtualbox" do |vb|
        vb.name = node_name
        vb.memory = 1024
        vb.cpus = 1
      end
      node.vm.hostname = node_name
      if BUILD_MODE == "BRIDGE"
        node.vm.network :public_network, ip: node_ip, bridge: get_bridge_adapter()
      else
        node.vm.network :private_network, ip: node_ip
        node.vm.network "forwarded_port", guest: 22, host: "#{2720 + index}"
      end
      provision_kubernetes_node node
    end
  end

  if BUILD_MODE == "BRIDGE"
    config.trigger.after :up do |trigger|
      trigger.name = "Post provisioner"
      trigger.ignore = [:destroy, :halt]
      trigger.ruby do |env, machine|
        if all_nodes_up()
          nodes = ["staticmaster"] + ["kubenode3", "kubenode4"]
          ips = nodes.map { |n| %x{vagrant ssh #{n} -c 'public-ip'}.chomp }
          hosts = ips.zip(nodes).map { |node_ip, n| "#{node_ip}  #{n}" }.join("\n")
          File.open("hosts.tmp", "w") { |file| file.write(hosts) }
          nodes.each do |node|
            system("vagrant upload hosts.tmp /tmp/hosts.tmp #{node}")
            system("vagrant ssh #{node} -c 'cat /tmp/hosts.tmp | sudo tee -a /etc/hosts'")
          end
          File.delete("hosts.tmp")
          puts "VM build complete!"
        else
          puts "Nothing to do here"
        end
      end
    end
  end
end
