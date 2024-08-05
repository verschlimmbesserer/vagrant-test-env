# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'yaml'

vagrant_hosts = ENV['VAGRANT_ENV'] ? ENV['VAGRANT_ENV'] : 'vagrant.yaml'
servers = YAML.load_file(File.join(__dir__, vagrant_hosts), aliases: true)

DEFAULT_BOX = "bento/ubuntu-22.04"
DEFAULT_VM_MEMORY = 1024
DEFAULT_VM_CPU = 1
VAGRANTFILE_API_VERSION = "2"

VagrantDNS::Config.listen = [[:udp, "192.168.122.1", 5300],[:tcp, "192.168.122.1", 5300]]

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

    provider = servers.fetch("provider", ["parallels", "vmware_desktop", "libvirt"])
    config.vm.provider provider.downcase

    config.vagrant.plugins = [{"vagrant-dns" => {"version" => "2.4.1"}}]

    servers["hosts"].each do |server|
        config.vm.define server["name"] do |srv|
            srv.vm.box = server.fetch("box", DEFAULT_BOX)
            srv.vm.box_version = server["box_version"] if server["box_version"]
            srv.vm.box_check_update = server["box_check_update"] if server["box_check_update"]
            srv.vm.hostname = server["name"]
            srv.dns.tld = servers["dns"]["tld"] ||= "vagrant.test"
            srv.vm.provider "vmware_desktop" do |vmware|
                vmware.gui = server["gui"] if server["gui"]
                vmware.linked_clone = server["linked_clone"]
                vmware.vmx["memsize"] = server["ram"] ||= DEFAULT_VM_MEMORY
                vmware.vmx["numvcpus"] = server["cpus"] ||= DEFAULT_VM_CPU
            end

            srv.vm.provider "parallels" do | prl |
                prl.name = server["name"]
                prl.linked_clone = server["linked_clone"] ||= false
                prl.update_guest_tools = server["update_guest_tools"] ||= false

                prl.memory = server["ram"] ||= DEFAULT_VM_MEMORY
                prl.cpus   = server["cpus"] ||= DEFAULT_VM_CPU

                if server["customize"]
                  server["customize"].each do |option, state|
                    prl.customize ["set", :id, "--#{option}", "#{state}"]
                  end
                end
            end

            srv.vm.provider :libvirt do |libvirt|
              libvirt.memory = server["ram"] ||= DEFAULT_VM_MEMORY
              libvirt.cpus = server["cpus"] ||= DEFAULT_VM_CPU
              libvirt.default_prefix = ""
            end


            if server["private_network"]
                server['private_network'].each do |private_network|
                  srv.vm.network "private_network",
                      type: private_network.fetch("type", nil),
                      ip: private_network["ip_private"],
                      auto_config: private_network.fetch("auto_config", nil)
                end
            end

            if server["public_network"]
                server['public_network'].each do |public_network|
                  srv.vm.network "public_network",
                          use_dhcp_assigned_default_route: public_network["use_dhcp_assigned_default_route"],
                          auto_config: public_network.fetch("auto_config", true),
                          bridge: public_network.fetch("bridge", DEFAULT_BRIDGE),
                          ip: public_network["ip_public"]
                end
            end
            # Port forwarding
            if server["ports"]
              server["ports"].each do |ports|
                srv.vm.network "forwarded_port",
                          guest: ports["guest"],
                          host: ports["host"],
                          auto_correct: true
              end
            end

            # Synced folders
            if server["syncDir"]
                server['syncDir'].each do |syncDir|
                    srv.vm.synced_folder syncDir['host'],
                               syncDir['guest'],
                               owner: syncDir["owner"],
                               group: syncDir["group"],
                               mount_options: ["dmode=#{syncDir["dmode"]}", "fmode=#{syncDir["fmode"]}"],
                               create: true
                end
            end

            # Disk configurations
            if server["disks"]
              server['disks'].each do |disks|
                srv.vm.disk :disk,
                      name: disks['name'],
                      size: disks['size']
              end
            end

            # Provisioning configurations (Cloud-init, Bash, Files, Ansible, SaltStack)
            if server["cloud_init"]
              srv.vm.cloud_init do |cloud_init|
                cloud_init.content_type = server["cloud_init"]["content_type"]
                cloud_init.path = server["cloud_init"]["path"]
              end
            end

            if server["bash"]
              srv.vm.provision :shell, path: server["bash"]
            end

            if server["files"]
              server["files"].each do |files|
                srv.vm.provision "file", source: files["source"], destination: files["destination"]
              end
            end

            if server["ansible"]
                srv.vm.provision :ansible do |ansible|
                    ansible.verbose = server["ansible"]["verbose"] if server["ansible"]["verbose"]
                    ansible.playbook = server["ansible"]["playbook"] if server["ansible"]["playbook"]
                    ansible.inventory_path = server["ansible"]["inventory_path"] if server["ansible"]["inventory_path"]
                    ansible.host_key_checking = server["ansible"]["host_key_checking"] if server["ansible"]["host_key_checking"]
                    ansible.limit = server["ansible"]["limit"] if server["ansible"]["limit"]
                end
            end

            if server["salt"]
                srv.vm.provision :salt do |salt|
                    salt.install_master = server["salt"]["install_master"] if server["salt"]["install_master"]
                    salt.install_type = server["salt"]["install_type"] || "stable"
                    salt.no_minion = server["salt"]["no_minion"] if server["salt"]["no_minion"]
                    salt.install_syncdir = server["salt"]["install_syncdir"] if server["salt"]["install_syncdir"]
                    salt.install_args = server["salt"]["install_args"] if server["salt"]["install_type"] == "git" && server["salt"]["install_args"] != "develop"
                    salt.always_install = server["salt"]["always_install"] if server["salt"]["always_install"]
                    salt.bootstrap_script = server["salt"]["bootstrap_script"] if server["salt"]["bootstrap_script"]
                    salt.bootstrap_options = server["salt"]["bootstrap_options"] if server["salt"]["bootstrap_options"]
                    salt.version = server["salt"]["version"] if server["salt"]["version"]
                    salt.python_version = server["salt"]["python_version"] if server["salt"]["python_version"]
                    salt.run_highstate = server["salt"]["run_highstate"] if server["salt"]["run_highstate"]
                    salt.colorize = server["salt"]["colorize"] if server["salt"]["colorize"]
                    salt.log_level = server["salt"]["log_level"] if server["salt"]["log_level"]
                    salt.verbose = server["salt"]["verbose"] if server["salt"]["verbose"]
                end

                srv.trigger.after [:up, :provision, :reload] do |tu|
                  tu.name = "accept all keys"
                  tu.info = "accept minion key on master"
                  tu.run = { inline: "vagrant ssh salt -- while sudo salt-key -l unaccepted | grep -q #{server["name"]}; do sudo /usr/bin/salt-key -y -A ; done"}
                end

                if server["name"] != "salt"
                  srv.trigger.after :destroy do |td|
                    td.name = "remove keys"
                    td.info = "remove minion key on master after destroy"
                    td.run = { inline: "vagrant ssh salt -- sudo /usr/bin/salt-key -y -d #{server["name"]}" }
                  end
                end
            end
        end
    end
end
