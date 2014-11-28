# -*- mode: ruby -*-
# # vi: set ft=ruby :

# We are using this
# http://blog.dzema.name/2014/03/24/provider-dependent-box-in-vagrant.html
# trick in order to select the correct box name for
# virtualbox or vmware_fusion in order to avoid having to alter
# the coreos build scripts.

require 'fileutils'

Vagrant.require_version ">= 1.6.0"

CLOUD_CONFIG_PATH = File.join(File.dirname(__FILE__), "user-data")
CONFIG = File.join(File.dirname(__FILE__), "config.rb")

# Defaults for config options defined in CONFIG
$num_instances = 1
$update_channel = "alpha"
$enable_serial_logging = false
$vb_gui = false
# $vb_memory = 1024
$vb_memory = 2048
$vb_cpus = 1

# Attempt to apply the deprecated environment variable NUM_INSTANCES to
# $num_instances while allowing config.rb to override it
if ENV["NUM_INSTANCES"].to_i > 0 && ENV["NUM_INSTANCES"]
  $num_instances = ENV["NUM_INSTANCES"].to_i
end

if File.exist?(CONFIG)
  require CONFIG
end

Vagrant.configure("2") do |config|
  # Every Vagrant virtual environment requires a box to build off of.
  # Our developers use different virtual machines to work (which can produce
  # bugs which are hard to reproduce but let's leave it for now) so we moved
  # box definition into configuration of specific provider.
  # When user runs `vagrant up` it will pick up correct box
  # from provider specific config.
  #
  # It is implemented with override object which is documented
  # here https://docs.vagrantup.com/v2/providers/configuration.html
  config.vm.provider :vmware_fusion do |wmf, override|
    override.vm.box = "coreos_developer_vagrant_vmware_fusion"
  end

  config.vm.provider :virtualbox do |vb, override|
    override.vm.box = "coreos_developer_virtualbox"
  end
  # config.vm.box = "coreos_developer_vagrant_vmware_fusion"
  # config.vm.box = "coreos_developer_virtualbox"

  # Cygnus: Commented this out to use our own custom images.
  # config.vm.box_version = ">= 308.0.1"
  # config.vm.box_url = "http://%s.release.core-os.net/amd64-usr/current/coreos_production_vagrant.json" % $update_channel
  # config.vm.box_url = "file://coreos_developer_vagrant_vmware_fusion.json"

  # config.vm.provider :vmware_fusion do |vb, override|
  #   override.vm.box_url = "http://%s.release.core-os.net/amd64-usr/current/coreos_production_vagrant_vmware_fusion.json" % $update_channel
  # end

  # Cygnus: Use this to set the box name,

  config.vm.provider :virtualbox do |v|
    # On VirtualBox, we don't have guest additions or a functional vboxsf
    # in CoreOS, so tell Vagrant that so it can be smarter.
    v.check_guest_additions = false
    v.functional_vboxsf     = false
  end

  # plugin conflict
  if Vagrant.has_plugin?("vagrant-vbguest") then
    config.vbguest.auto_update = false
  end

  (1..($num_instances*2)).step(2).each do |i|
    config.vm.define vm_name = "core-%02d" % i do |config|
      config.vm.hostname = vm_name

      if $enable_serial_logging
        logdir = File.join(File.dirname(__FILE__), "log")
        FileUtils.mkdir_p(logdir)

        serialFile = File.join(logdir, "%s-serial.txt" % vm_name)
        FileUtils.touch(serialFile)

        config.vm.provider :vmware_fusion do |v, override|
          v.vmx["serial0.present"] = "TRUE"
          v.vmx["serial0.fileType"] = "file"
          v.vmx["serial0.fileName"] = serialFile
          v.vmx["serial0.tryNoRxLoss"] = "FALSE"
        end

        config.vm.provider :virtualbox do |vb, override|
          vb.customize ["modifyvm", :id, "--uart1", "0x3F8", "4"]
          vb.customize ["modifyvm", :id, "--uartmode1", serialFile]
        end
      end

      if $expose_docker_tcp
        config.vm.network "forwarded_port", guest: 2375, host: ($expose_docker_tcp + i - 1), auto_correct: true
      end

      config.vm.provider :vmware_fusion do |vb|
        vb.gui = $vb_gui
        vb.vmx["memsize"] = $vb_memory
      end

      config.vm.provider :virtualbox do |vb|
        vb.gui = $vb_gui
        vb.memory = $vb_memory
        vb.cpus = $vb_cpus
      end

      ip1 = "172.17.8.#{(i)+100}"
      ip2 = "172.17.8.#{(i+1)+100}"
      config.vm.network :private_network, ip: ip1
      config.vm.network :private_network, ip: ip2

      # Uncomment below to enable NFS for sharing the host machine into the coreos-vagrant VM.
      #config.vm.synced_folder ".", "/home/core/share", id: "core", :nfs => true, :mount_options => ['nolock,vers=3,udp']

      if File.exist?(CLOUD_CONFIG_PATH)
        config.vm.provision :file, :source => "#{CLOUD_CONFIG_PATH}", :destination => "/tmp/vagrantfile-user-data"
        config.vm.provision :shell, :inline => "mv /tmp/vagrantfile-user-data /var/lib/coreos-vagrant/", :privileged => true
      end

    end
  end
end
