# -*- mode: ruby -*-
# vi: set ft=ruby :

def inventory_discovery (fileVInventory, fileAInventory, ipMap, groups)

  # set proxy if defined in ENV
  proxy = ""
  if (ENV.has_key?('HTTP_PROXY'))
    proxy = ENV['HTTP_PROXY']
  end
  if (ENV.has_key?('http_proxy'))
    proxy = ENV['http_proxy']
  end

  # Initialize variable to keep track of group name
  groupName = ""
  nodeCount = 0

  # Read Vagrant inventory file
  File.open(fileVInventory, "r") do |f|

    # For each line in the inventory file
    f.each_line do |line|

      # Remove comments
      line = line.gsub(/#.*/, '')

      # If the line has a group name
      if (line =~ /\[(.*)\]/)

        # Remember the group name
	groupName = $1
	nodeCount = 1 

	# If this group name is not in the hash, then set it up and use defaults
	if (!groups.has_key?(groupName))
	  groups[groupName] = {}
          groups["default"].each do |parameterName, parameter|
	    groups[groupName][parameterName] = parameter
          end
          if (!(groups[groupName][":kubernetes_stem"] == "true") && (proxy != ""))
            groups[groupName][":proxy"] = proxy
          end
        end

      # If the line contains an IP address
      elsif (line =~ /:ip\s+([0-9]+\.[0-9]+\.[0-9]+\.[0-9]+)/)

        ipaddrs = line.split(" ")

	# Initialize IP address list if needed
	if (!groups[groupName].has_key?(":ip"))
	  groups[groupName][":ip"] = []
	end

        # Generate node name
        nodeName = groupName + groups[groupName][":format"] % nodeCount
        nodeCount = nodeCount + 1

	# Record critical information for IP address
        groups[groupName][":ip"] << $1
	if (!ipMap.has_key?($1))
	  ipMap[$1] = {}
          if (ipaddrs.length == 3)
            ipMap[$1][":extip"] = ipaddrs[2]
          end
          ipMap[$1][":hostname"] = nodeName
	  ipMap[$1][":box"] = groups[groupName][":box"]
	  ipMap[$1][":memory"] = 0.0
	  ipMap[$1][":cores"] = 0.0
	  if (!(groups[groupName][":kubernetes_stem"] == "true"))
	    if (groups[groupName].has_key?(":proxy"))
	      ipMap[$1][":proxy"] = groups[groupName][":proxy"]
	    end
          else
              ipMap[$1][":kubernetes_stem"] = groups[groupName][":kubernetes_stem"]
          end
        end
        ipMap[$1][":memory"] += groups[groupName][":memory"].to_f
        ipMap[$1][":cores"] += groups[groupName][":cores"].to_f

      # Generic group information
      elsif (line =~ /(:\S+)\s+(\S+)/)

        groups[groupName][$1] = $2


      end

    end

  end

  # Write Ansible inventory file
  File.open(fileAInventory, "w") do |f|

    # Find current Vagrant home directory
    vagrantHome = File.dirname(__FILE__)

    # Set up the group vars for kubernetes.  You must use kubernetes and not 
    # mesos_proxy since this will break the proxy env that vagrant
    # sets up.
    top_level_group = 'kubernetes'
    f.puts "[" + top_level_group + ":children]"

    flannel_var = '' 
    if (groups['default'].has_key?(':flannel_interface'))
      flannel_var = ' flannel_interface=' + groups['default'][':flannel_interface']
    end
    provider = 'virtualbox'
    if (groups['default'].has_key?(':provider'))
      provider = groups['default'][':provider']
    end
    puts "provider " + provider

    groups.each do |groupName, group|
        if groupName == 'default'
            next
        end
        f.puts groupName
    end

    # For each group...
    groups.each do |groupName, group|

      # Print out group name
      if (groupName != "default")
        f.puts "[" + groupName + "]"
      end

      if (group.has_key?(":ip"))

        # For each ip address in the group...
        group[":ip"].each do |ip|

	  multiple_registry_hosts = ""
          if (group.has_key?(":multiple_registry_hosts"))
            multiple_registry_hosts = " multiple_registry_hosts=" + group[":multiple_registry_hosts"]
          end

          use_vault_opt = ""
	  if (group.has_key?(":use_vault"))
            use_vault_opt = " use_vault=" + group[":use_vault"]
          end

	  # Generate the inventory entry
          # you need ansible_ssh_user for < 2.0 ansible, which most setups in picasso group
          # is using currently
          line = ipMap[ip][":hostname"] + " ansible_ssh_host=" + ip +
	         " ansible_ssh_user=vagrant ansible_user=vagrant ansible_ssh_private_key_file=" +
                 vagrantHome + "/.vagrant/machines/" + ipMap[ip][":hostname"] + "/" + provider + "/private_key" +
                 multiple_registry_hosts + flannel_var + use_vault_opt

          f.puts line

        end

      end

      f.puts ""

    end

  end

end


def deploy_nodes(config, ipMap, groups)

  # If we have IP addresses in the group
  if (!ipMap.empty?)

    # For each ip address...
    ipMap.each do |ip, ipData|

      # Configure the node
      config.vm.define ipData[":hostname"] do |node|

        # Point to base VM
        node.vm.box = ipData[":box"]
        node.vm.provider :vmware_driver do |provider|
          node.vm.synced_folder "./", "/vagrant", disabled: true
        end

        # Define hostname (Note: This does not work for Ubuntu Wily...)
        node.vm.hostname = ipData[":hostname"]

        # Set up networking
        node.vm.network "private_network", ip: ip
        if  (ipData.has_key?(":extip"))
            node.vm.network "private_network", ip: ipData[":extip"]
        end

	if (ipData.has_key?(":kubernetes_stem"))
            node.vm.provision :hosts, :add_localhost_hostnames => false, :autoconfigure => false
        else
            node.vm.provision :hosts, :sync_hosts => true, :add_localhost_hostnames => false
        end

	# If we are using a proxy...
	if (ipData.has_key?(":proxy"))

          # Generate the global no_proxy environment variable
          noProxyList = "localhost,127.0.0.1" 
          ipMap.each do |localIP, localIPData|
            noProxyList = noProxyList + "," + localIP
            noProxyList = noProxyList + "," + ipMap[localIP][":hostname"]
          end
 
	  # Set up proxy on VM
          node.proxy.http = ipData[":proxy"]
          node.proxy.https = ipData[":proxy"]
          node.proxy.ftp = ipData[":proxy"]
          node.proxy.no_proxy = noProxyList

	end

        # Set up VM memory and CPU sizes
        node.vm.provider "virtualbox" do |vb|
          vb.memory = ipData[":memory"].ceil
          vb.cpus = ipData[":cores"].ceil
        end

        node.vm.provider "vmware_fusion" do |v|
          v.vmx["memsize"] = ipData[":memory"].ceil
          v.vmx["numvcpus"] = ipData[":cores"].ceil 
end

	# Add users public key to vagrant VM
	publicKey = ""
        if (ENV.has_key?('HOME'))
          publicKey = ENV['HOME'] + "/.ssh/id_rsa.pub"
        end
	if (File.exists?(publicKey))
	  node.vm.provision "file", source: publicKey, destination: "/tmp/id_rsa.pub"
          node.vm.provision "shell", inline: "cat /tmp/id_rsa.pub >> /home/vagrant/.ssh/authorized_keys"
          node.vm.provision "shell", inline: "rm /tmp/id_rsa.pub"
        end

	# Set up ansbile provisioner if last IP address
	if ((ip == ipMap.keys.last) && (groups["default"][":run_ansible_playbook"] == "true"))
          puts "Ansible playbook enabled"
          node.vm.provision "ansible" do |ansible|
            ansible.verbose = "v"
            ansible.sudo = "true"
            ansible.inventory_path = ".vagrant/ansible-inventory"
            if (groups["default"].has_key?(":multiple_registry_hosts"))
              ansible.extra_vars = { multiple_registry_hosts: groups["default"][":multiple_registry_hosts"] }
            end
            ansible.playbook = "../kubernetes-cluster.yml"
            ansible.limit = "all" 
          end
	elsif (ip == ipMap.keys.last)
          puts "Ansible playbook disabled"
	end

      end

    end

  end

end


Vagrant.configure(2) do |config|

  # Check for plugins
  unless Vagrant.has_plugin?("vagrant-hosts")
    puts 'vagrant: \'vagrant-hosts\' plugin not installed!'
    abort 'vagrant: Please run \'vagrant plugin install vagrant-hosts\''
  end
  unless Vagrant.has_plugin?("vagrant-proxyconf")
    puts 'vagrant: \'vagrant-proxyconf\' plugin not installed!'
    abort 'vagrant: Please run \'vagrant plugin install vagrant-proxyconf\''
  end

  # Initialize ipMap to be a hash
  ipMap = {}

  # Initiaize groups to be a hash and add default group
  groups = {}
  groups["default"] = {}

  # Look for customized cluster-inventory or else use example
  clusterInventoryFile = ".vagrant/cluster-inventory"
  if (!File.exists?(clusterInventoryFile))
    puts "vagrant: No '.vagrant/cluster-inventory' found.  (See README.)"
    puts "vagrant: Using 'cluster-inventory.example'"
    clusterInventoryFile = "cluster-inventory.example"
  end

  # Read ansible inventory file
  inventory_discovery(
    clusterInventoryFile,
    ".vagrant/ansible-inventory",
    ipMap, groups)

  # Create clusters based on groups found in inventory
  deploy_nodes(config, ipMap, groups)

end
