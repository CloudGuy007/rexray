# -*- mode: ruby -*-
# vi: set ft=ruby :

# please set the volume_path variable to a valid directory
# for storing virtualbox volumes on the local, host system
$volume_path = "/Users/akutz/VirtualBox/Volumes"

# node info
$node0_name = "node0"
$node0_ip   = "192.168.56.10"
$node0_mem  = "1024"

$node1_name = "node1"
$node1_ip   = "192.168.56.11"
$node1_mem  = "512"

# Golang information
$goos   = "linux"
$goarch = "amd64"
$gover  = "1.6.2"
$gotgz  = "go#{$gover}.#{$goos}-#{$goarch}.tar.gz"
$gourl  = "https://storage.googleapis.com/golang/#{$gotgz}"
$gopath = "/opt/go"

# the script to provision golang
$provision_golang = <<SCRIPT
echo installing go#{$gover}.#{$goos}-#{$goarch}
wget -q #{$gourl} && tar -C /usr/local -xzf #{$gotgz} && mkdir -p #{$gopath}
SCRIPT

# the script to provision docker
$provision_docker = <<SCRIPT
apt-get update
apt-get install apt-transport-https ca-certificates
apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 \
            --recv-keys 58118E89F3A912897C070ADBF76221572C52609D
echo 'deb https://apt.dockerproject.org/repo ubuntu-trusty main' > \
     /etc/apt/sources.list.d/docker.list
apt-get update
apt-get purge lxc-docker
apt-get install linux-image-extra-$(uname -r)
apt-get install apparmor
apt-get update
apt-get install -y docker-engine
usermod -a -G docker vagrant
service docker start
docker run hello-world
SCRIPT

# go-bindata info
$go_bindata_dir = "#{$gopath}/src/github.com/jteeuwen/go-bindata"
$go_bindata_url = "https://github.com/akutz/go-bindata"
$go_bindata_ref = "feature/md5checksum"

# the script to build go-bindata
$build_go_bindata = <<SCRIPT
mkdir -p #{$go_bindata_dir}
cd #{$go_bindata_dir}
git clone #{$go_bindata_url} .
git checkout #{$go_bindata_ref}
go get ./...
go install ./...
SCRIPT

# rex-ray repo and branch information
$rexray_dir = "#{$gopath}/src/github.com/emccode/rexray"
$rexray_url = "https://github.com/akutz/rexray"
$rexray_ref = "release/0.4.0-rc4"
$rexray_bin = "#{$gopath}/bin/rexray"
$rexray_cfg = "/etc/rexray/config.yml"

# the script to build rex-ray
$build_rexray = <<SCRIPT
mkdir -p #{$rexray_dir}
cd #{$rexray_dir}
git clone #{$rexray_url} .
git checkout #{$rexray_ref}
make deps
make
SCRIPT

# the script to write node0's rex-ray config file. the 'virtualbox.volumePath'
# property should be replaced with a valid directory path on the virtualbox
# host system
$write_rexray_config_node0 = <<SCRIPT
mkdir -p $(dirname #{$rexray_cfg})
cat << EOF > #{$rexray_cfg}
rexray:
  logLevel: warn
libstorage:
  host:     tcp://127.0.0.1:7979
  embedded: true
  service:  virtualbox
  server:
    endpoints:
      public:
        address: tcp://:7979
    services:
      virtualbox:
        driver: virtualbox
virtualbox:
  volumePath: #{$volume_path}
EOF
SCRIPT

# the script to write node1's rex-ray config file. the 'virtualbox.volumePath'
# property should be replaced with a valid directory path on the virtualbox
# host system
$write_rexray_config_node1 = <<SCRIPT
mkdir -p $(dirname #{$rexray_cfg})
cat << EOF > #{$rexray_cfg}
rexray:
  logLevel: warn
libstorage:
  host:    tcp://#{$node0_ip}:7979
  service: virtualbox
EOF
SCRIPT

# init the environment variables used when building go source
$build_env_vars = Hash[
    "GOPATH" => $gopath,
    "PATH" => "#{$gopath}/bin:/usr/local/go/bin:#{ENV['PATH']}"
]

# node_dir returns the directory for a given node
def node_dir(name)
    return "#{File.dirname(__FILE__)}/.vagrant/machines/#{name}"
end

# is_first_up returns a flag indicating whether or not this is the first time
# 'vagrant up' has been called on a specific node
def is_first_up(name)
  return Dir.glob("#{node_dir(name)}/*/id").empty?
end

# init_node initializes the node information
def init_node(node, name, ip)
  node.vm.box = "ubuntu/trusty64"
  node.vm.hostname = name
  node.vm.network :private_network, ip: ip
end

# init_virtualbox initializes the virtualbox settings for a VM
def init_virtualbox(vb, ram)
  # set the VM's RAM size. must be at least 1GB.
  # see https://github.com/beego/wetalk/issues/32
  # https://groups.google.com/forum/#!topic/golang-nuts/0qUdADqqsDs
  vb.memory = ram

  # renamed the SATA controller to be the default name for a VirtualBox
  # SATA controller, `SATA`
  vb.customize ["storagectl", :id, "--name", "SATAController", "--rename", "SATA"]

  # set the SATA controller's port count so it's greater than 1. if this
  # step is omitted it is not possible to attach new volumes to this VM.
  vb.customize ["storagectl", :id, "--name", "SATA", "--portcount", "25"]

  # enable NAT DNS
  vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]

  # make sure the first NIC has a "random" MAC address to ensure it is not
  # the same MAC address as the other node
  vb.customize ["modifyvm", :id, "--macaddress1", "auto"]

end

Vagrant.configure("2") do |config|

  # configure node0
  config.vm.define $node0_name do |node|

    # initialize the node information
    init_node node, $node0_name, $node0_ip

    # only proceed if this is the first time 'vagrant up' has been called
    # on this node
    if is_first_up node.vm.hostname

      # initialize virtualbox
      node.vm.provider :virtualbox do |vb|
        init_virtualbox vb, $node0_mem
      end

      # provision docker - don't use docker provisioner - see
      # https://github.com/mitchellh/vagrant/issues/7161
      node.vm.provision "shell" do |s|
        s.name =   "docker"
        s.inline = $provision_docker
      end

      # provision golang
      node.vm.provision "shell" do |s|
        s.name   = "golang"
        s.inline = $provision_golang
      end

       # build go-bindata
       node.vm.provision "shell" do |s|
         s.name   = "go-bindata"
         s.env    = $build_env_vars
         s.inline = $build_go_bindata
       end

      # build rex-ray
      node.vm.provision "shell" do |s|
        s.name   = "build rex-ray"
        s.env    = $build_env_vars
        s.inline = $build_rexray
      end

      # write rex-ray config file
      node.vm.provision "shell" do |s|
        s.name       = "config rex-ray"
        s.inline     = $write_rexray_config_node0
      end

      # install rex-ray
      node.vm.provision "shell" do |s|
        s.name   = "rex-ray install"
        s.inline = "#{$rexray_bin} install"
      end

      # start rex-ray as a service
      node.vm.provision "shell" do |s|
        s.name   = "start rex-ray"
        s.inline = "/etc/init.d/rexray start"
      end

    end # if is_first_up node.vm.hostname

    # list volume mapping with rex-ray to verify configuration
    node.vm.provision "shell", run: "always" do |s|
      s.name       = "rex-ray volume map"
      s.privileged = false
      s.inline     = "#{$rexray_bin} volume map"
    end

  end # configure node0

  # configure node1
  config.vm.define $node1_name do |node|

    # initialize the node information
    init_node node, $node1_name, $node1_ip

    # only proceed if this is the first time 'vagrant up' has been called
    # on this node
    if is_first_up node.vm.hostname

      # initialize virtualbox
      node.vm.provider :virtualbox do |vb|
        init_virtualbox vb, $node1_mem
      end

      # provision docker - don't use docker provisioner - see
      # https://github.com/mitchellh/vagrant/issues/7161
      node.vm.provision "shell" do |s|
        s.name =   "docker"
        s.inline = $provision_docker
      end

      # copy node0's private ssh key to node1 so node1 can ssh to node0 without
      # being prompted for a password
      node.vm.provision              "file",
                        source:      "#{node_dir($node0_name)}" +
                                     "/virtualbox/private_key",
                        destination: "$HOME/.ssh/#{$node0_name}.key"

      # scp rex-ray from node0 to node1
      node.vm.provision "shell" do |s|
        s.name =   "scp rexray"
        s.inline = "mkdir -p $(dirname #{$rexray_bin}); " +
                   "scp -q -i /home/vagrant/.ssh/#{$node0_name}.key " +
                   "-o StrictHostKeyChecking=no " +
                   "vagrant@#{$node0_ip}:#{$rexray_bin} #{$rexray_bin}"
      end

      # write rex-ray config file
      node.vm.provision "shell" do |s|
        s.name       = "config rex-ray"
        s.inline     = $write_rexray_config_node1
      end

      # install rex-ray
      node.vm.provision "shell" do |s|
        s.name   = "rex-ray install"
        s.inline = "#{$rexray_bin} install"
      end

      # start rex-ray as a service
      node.vm.provision "shell" do |s|
        s.name   = "start rex-ray"
        s.inline = "/etc/init.d/rexray start"
      end

    end # if is_first_up node.vm.hostname

    # list volume mapping with rex-ray to verify configuration
    node.vm.provision "shell", run: "always" do |s|
      s.name       = "rex-ray volume map"
      s.privileged = false
      s.inline     = "#{$rexray_bin} volume map"
    end

  end # configure node1

end
