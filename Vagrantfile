# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'fileutils'
require 'net/http'
require 'open-uri'
require 'json'

class Module
  def redefine_const(name, value)
    __send__(:remove_const, name) if const_defined?(name)
    const_set(name, value)
  end
end

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
end

if OS.windows?
  abort "You're running on windows, an unsupported host platform. Exiting..."
end

def getK8Sreleases()
  url = "https://api.github.com/repos/GoogleCloudPlatform/kubernetes/releases"
  json =  JSON.load(open(url))
  return json.collect { |item|
    if KUBERNETES_ALLOW_PRERELEASES
      item['tag_name'].gsub('v', '')
    else
      item['tag_name'].gsub('v', '') if item['prerelease'] == false
    end
  }.compact
end

def checkResponse(host, port, expectValidResponse = true, description = "",
  attempts = 50)
  hasResponse = false
  target = "http://#{host}:#{port}"
  txt  = "#{target} "
  txt += "(#{description})" if description
  j, uri, res = 0, URI(target), nil

  loop do
    j += 1
    begin
      res = Net::HTTP.get_response(uri)
      rescue Net::HTTPBadResponse
        hasResponse = true if not expectValidResponse
      rescue
        sleep 10
    end
    break if hasResponse or res.is_a? Net::HTTPSuccess or j >= attempts
  end
  mesg = "WARNING: something going wrong - #{txt} taking too long to respond."
  puts "#{mesg}" if j >= attempts
end

def createFromTemplate(template, destination, hostname = "")
  File.delete(destination) if File.exist?(destination)
  data = File.read(template)
  data.gsub!( /__(.*?)__/ ) {
    begin
      eval("#{$1}")
    end
  }
  File.open(destination, "w") do |f|
   f.write(data)
  end
end

required_plugins = %w(vagrant-triggers)
required_plugins.each do |plugin|
  need_restart = false
  unless Vagrant.has_plugin? plugin
    system "vagrant plugin install #{plugin}"
    need_restart = true
  end
  exec "vagrant #{ARGV.join(' ')}" if need_restart
end

# Vagrantfile API/syntax version. Don't touch unless you know what you're doing!
VAGRANTFILE_API_VERSION = "2"
Vagrant.require_version ">= 1.6.0"

MASTER_YAML = File.join(File.dirname(__FILE__), "master.yaml")
NODE_YAML = File.join(File.dirname(__FILE__), "node.yaml")
SSL_FILE = File.join(File.dirname(__FILE__), "kube-serviceaccount.key")

DOCKERCFG = File.expand_path(ENV['DOCKERCFG'] || "~/.dockercfg")
KUBERNETES_ALLOW_PRERELEASES = (ENV['KUBERNETES_ALLOW_PRERELEASES'] =~ (/(true|t|yes|y|1)$/i) && true) || false
KUBERNETES_VERSION = ENV['KUBERNETES_VERSION'] || 'latest'
if KUBERNETES_VERSION == "latest"
  Object.redefine_const(:KUBERNETES_VERSION, getK8Sreleases[0])
end
# for backward compat ...
RELEASE = KUBERNETES_VERSION

CHANNEL = ENV['CHANNEL'] || 'alpha'
if CHANNEL != 'alpha'
  puts "============================================================================="
  puts "As this is a fastly evolving technology CoreOS' alpha channel is the only one"
  puts "expected to behave reliably. While one can invoke the beta or stable channels"
  puts "please be aware that your mileage may vary a whole lot."
  puts "So, before submitting a bug, in this project, or upstreams (either kubernetes"
  puts "or CoreOS) please make sure it (also) happens in the (default) alpha channel."
  puts "============================================================================="
end

COREOS_VERSION = ENV['COREOS_VERSION'] || 'latest'
upstream = "http://#{CHANNEL}.release.core-os.net/amd64-usr/#{COREOS_VERSION}"
if COREOS_VERSION == "latest"
  upstream = "http://#{CHANNEL}.release.core-os.net/amd64-usr/current"
  url = "#{upstream}/version.txt"
  Object.redefine_const(:COREOS_VERSION,
    open(url).read().scan(/COREOS_VERSION=.*/)[0].gsub('COREOS_VERSION=', ''))
end

NUM_INSTANCES = ENV['NUM_INSTANCES'] || 2

MASTER_MEM = ENV['MASTER_MEM'] || 1024
MASTER_CPUS = ENV['MASTER_CPUS'] || 1

NODE_MEM= ENV['NODE_MEM'] || 2048
NODE_CPUS = ENV['NODE_CPUS'] || 1

SERIAL_LOGGING = (ENV['SERIAL_LOGGING'].to_s.downcase == 'true')
GUI = (ENV['GUI'].to_s.downcase == 'true')

BASE_IP_ADDR = ENV['BASE_IP_ADDR'] || "172.17.8"

DNS_REPLICAS = ENV['DNS_REPLICAS'] || 1
DNS_DOMAIN = ENV['DNS_DOMAIN'] || "cluster.local"
DNS_UPSTREAM_SERVERS = ENV['DNS_UPSTREAM_SERVERS'] || "8.8.8.8:53,8.8.4.4:53"

# 'vagrant' taken out bellow due to 0.18.0 breakage
CLOUD_PROVIDER = ENV['CLOUD_PROVIDER'].to_s.downcase
validCloudProviders = [ 'gce', 'gke', 'aws', 'azure', 'vsphere',
  'libvirt-coreos', 'juju' ]
Object.redefine_const(:CLOUD_PROVIDER,
  '') unless validCloudProviders.include?(CLOUD_PROVIDER)

ETCD_SEED_CLUSTER = "master=http://#{BASE_IP_ADDR}.#{1+100}:2380"

# Read YAML file with mountpoint details
MOUNT_POINTS = YAML::load_file('synced_folders.yaml')

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  # always use Vagrants' insecure key
  config.ssh.insert_key = false
  config.ssh.forward_agent = true

  config.vm.box = "coreos-#{CHANNEL}"
  config.vm.box_version = "= #{COREOS_VERSION}"
  config.vm.box_url = "#{upstream}/coreos_production_vagrant.json"

  ["vmware_fusion", "vmware_workstation"].each do |vmware|
    config.vm.provider vmware do |v, override|
      override.vm.box_url = "#{upstream}/coreos_production_vagrant_vmware_fusion.json"
    end
  end

  config.vm.provider :parallels do |vb, override|
    override.vm.box = "AntonioMeireles/coreos-#{CHANNEL}"
    override.vm.box_url = "https://vagrantcloud.com/AntonioMeireles/coreos-#{CHANNEL}"
  end

  config.vm.provider :virtualbox do |v|
    # On VirtualBox, we don't have guest additions or a functional vboxsf
    # in CoreOS, so tell Vagrant that so it can be smarter.
    v.check_guest_additions = false
    v.functional_vboxsf     = false
  end
  config.vm.provider :parallels do |p|
    p.update_guest_tools = false
    p.check_guest_tools = false
  end

  # plugin conflict
  config.vbguest.auto_update = false if Vagrant.has_plugin?("vagrant-vbguest")

  (1..(NUM_INSTANCES.to_i + 1)).each do |i|
    if i == 1
      hostname = "master"
      cfg = MASTER_YAML
      memory = MASTER_MEM
      cpus = MASTER_CPUS
      MASTER_IP="#{BASE_IP_ADDR}.#{i+100}"
    else
      hostname = "node-%02d" % (i - 1)
      cfg = NODE_YAML
      memory = NODE_MEM
      cpus = NODE_CPUS
    end

    config.vm.define vmName = hostname do |kHost|
      kHost.vm.hostname = vmName

      # suspend / resume is hard to be properly supported because we have no way
      # to assure the fully deterministic behavior of whatever is inside the VMs
      # when faced with XXL clock gaps... so we just disable this functionality.
      kHost.trigger.reject [:suspend, :resume] do
        info "'vagrant suspend' and 'vagrant resume' are disabled."
        info "- please do use 'vagrant halt' and 'vagrant up' instead."
      end
      # simpler / more reliable
      kHost.trigger.reject :reload do
        info "'vagrant reload' is disabled."
        info "- please do use 'vagrant halt' and 'vagrant up' instead."
      end

      # vagrant-triggers has no concept of global triggers so to avoid having
      # then to run as many times as the total number of VMs we only call them
      # in the master (re: emyl/vagrant-triggers#13)...
      if vmName == "master"
        kHost.trigger.before [:up, :provision] do
          info "regenerating kubLocalSetup"
          tmpf = "#{Dir.pwd}/kubLocalSetup"
          createFromTemplate("#{tmpf}.tmpl", tmpf)
          File.chmod(0755, tmpf)

          info "making sure localhosts' 'kubectl' matches what we just booted..."
          system "./kubLocalSetup install"
        end

        kHost.trigger.after [:up, :resume] do
          info "making sure ssh agent has the default vagrant key..."
          system "ssh-add ~/.vagrant.d/insecure_private_key"

          info "making sure local fleetctl won't misbehave with old staled data..."
          info "(wiping old ~/.fleetctl/known_hosts)"
          kh = "#{Dir.pwd}/.fleetctl/known_hosts"
          File.delete(kh) if File.exist?(kh)
        end

        kHost.trigger.after [:up] do
          info "waiting for the master to be ready..."
          checkResponse(MASTER_IP, 8080, true, "kubernetes' master")

          info "configuring k8s internal dns service"
          rc = "#{Dir.pwd}/defaultServices/dns/skydns-rc.yaml"
          createFromTemplate("#{rc}.in", rc)
          system <<-EOT.prepend("\n\n") + "\n"
            $(./kubLocalSetup shellinit)
            kubectl create -f defaultServices/dns/skydns-rc.yaml
            kubectl create -f defaultServices/dns/skydns-svc.yaml.in
          EOT

          info "configuring k8s internal monitoring tools"
          system <<-EOT.prepend("\n\n") + "\n"
            $(./kubLocalSetup shellinit)
            kubectl create -f defaultServices/cluster-monitoring/grafana-service.yaml
            kubectl create -f defaultServices/cluster-monitoring/influxdb-service.yaml
            kubectl create -f defaultServices/cluster-monitoring/heapster-controller.yaml
            kubectl create -f defaultServices/cluster-monitoring/influxdb-grafana-controller.yaml
          EOT
        end

        kHost.trigger.after [:up, :resume] do
          info "============================================================================="
          info ""
          info "  please don't forget to run '$(./kubLocalSetup shellinit)' "
          info "                                                                             "
          info "  see the documentation at"
          info "    https://github.com/AntonioMeireles/kubernetes-vagrant-coreos-cluster/    "
          info ""
          info "============================================================================="
        end
      end

      if vmName != "master"
        kHost.trigger.after [:up] do
          info "waiting for #{hostname} (#{BASE_IP_ADDR}.#{i+100}) to be ready..."
          checkResponse("#{BASE_IP_ADDR}.#{i+100}", 10250, false,
            "#{hostname} (#{BASE_IP_ADDR}.#{i+100})")
        end
      end

      if SERIAL_LOGGING
        logdir = File.join(File.dirname(__FILE__), "log")
        FileUtils.mkdir_p(logdir)

        serialFile = File.join(logdir, "#{vmName}-serial.txt")
        FileUtils.touch(serialFile)

        ["vmware_fusion", "vmware_workstation"].each do |vmware|
          kHost.vm.provider vmware do |v, override|
            v.vmx["serial0.present"] = "TRUE"
            v.vmx["serial0.fileType"] = "file"
            v.vmx["serial0.fileName"] = serialFile
            v.vmx["serial0.tryNoRxLoss"] = "FALSE"
          end
        end
        kHost.vm.provider :virtualbox do |vb, override|
          vb.customize ["modifyvm", :id, "--uart1", "0x3F8", "4"]
          vb.customize ["modifyvm", :id, "--uartmode1", serialFile]
        end
        # supported since vagrant-parallels 1.3.7
        # https://github.com/Parallels/vagrant-parallels/issues/164
        kHost.vm.provider :parallels do |v|
          v.customize("post-import",
            ["set", :id, "--device-add", "serial", "--output", serialFile])
          v.customize("pre-boot",
            ["set", :id, "--device-set", "serial0", "--output", serialFile])
        end
      end

      ["vmware_fusion", "vmware_workstation", "virtualbox"].each do |h|
        kHost.vm.provider h do |vb|
          vb.gui = GUI
        end
      end
      ["parallels", "virtualbox"].each do |h|
        kHost.vm.provider h do |n|
          n.memory = memory
          n.cpus = cpus
        end
      end

      kHost.vm.network :private_network, ip: "#{BASE_IP_ADDR}.#{i+100}"
      # you can override this in synced_folders.yaml
      kHost.vm.synced_folder ".", "/vagrant", disabled: true

      begin
        MOUNT_POINTS.each do |mount|
          mount_options = ""
          disabled = false
          nfs =  true
          mount_options = mount['mount_options'] if mount['mount_options']
          disabled = mount['disabled'] if mount['disabled']
          nfs = mount['nfs'] if mount['nfs']

          if File.exist?(File.expand_path("#{mount['source']}"))
            if mount['destination']
              kHost.vm.synced_folder "#{mount['source']}", "#{mount['destination']}",
                id: "#{mount['name']}",
                disabled: disabled,
                mount_options: ["#{mount_options}"],
                nfs: nfs
            end
          end
        end
      rescue
      end

      if File.exist?(DOCKERCFG)
        kHost.vm.provision :file, run: "always",
         :source => "#{DOCKERCFG}", :destination => "/home/core/.dockercfg"

        kHost.vm.provision :shell, run: "always" do |s|
          s.inline = "cp /home/core/.dockercfg /root/.dockercfg"
          s.privileged = true
        end
      end

      if File.exist?(SSL_FILE)
        kHost.vm.provision :file, :source => "#{SSL_FILE}", :destination => "/home/core/kube-serviceaccount.key"
      end

      if File.exist?(cfg)
        user_data = "#{cfg}.#{hostname}.xxx"
        createFromTemplate(cfg, user_data, hostname)
        kHost.vm.provision :file, :source => "#{user_data}", :destination => "/tmp/vagrantfile-user-data"
        kHost.vm.provision :shell, :privileged => true,
        inline: <<-EOF
          mv /tmp/vagrantfile-user-data /var/lib/coreos-vagrant/
        EOF
      end

      kHost.trigger.after [:halt, :destroy] do
        # remove files created from templates...
        cruft = [ "#{cfg}.#{hostname}.xxx" ]
        if vmName == "master"
          cruft += [
            "#{Dir.pwd}/defaultServices/dns/skydns-rc.yaml",
            "#{Dir.pwd}/kubLocalSetup",
          ]
        end
        cruft.each do |f|
          File.delete(f) if File.exist?(f)
        end
      end
    end
  end
end
