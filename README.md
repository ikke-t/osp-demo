# osp-demo
Red Hat OpenStack Platform demo environment setup 

This is my memo for creating OpenStack demo env to run on my laptop. At the time it built  OSP8 B5. You need to replace 'YOUR_RHN_USERNAME', 'YOUR_RHN_PASSWD' and 'YOUR_RHN_POOL' below, otherwise this is copy paste operation to create OpenStack all in one demo machine. This was done on beta, thus the yum.repo. Normally with Red Hat repos you could just have the OSP repos in enabled repos. Or if you use RDO, point that to RDO.



# Install RHEL7 image for vagrant
I dunno the history of this image, it would be nice to get more fresh one:

```
vagrant box add --name rhel-server-7 http://file.bos.redhat.com/~lawhite/rhel-server-virtualbox-7.1-1.x86_64.box
```

# Create Vagrant file

```
mkdir ~/vagrant/osp
cd ~/vagrant/osp

cat > Vagrantfile <<EOF
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.box = "rhel-server-7"
  config.ssh.insert_key = false
  config.vm.define "osaio" do |rhel|
          rhel.vm.hostname   = "osaio.osnet"
          rhel.vm.box     = "rhel-server-7"

          # mgt network
          rhel.vm.network :private_network, ip: "192.168.56.3", :netmask => "255.255.255.0"

          # public ips
          rhel.vm.network :private_network, ip: "192.168.30.3", :netmask => "255.255.255.0"

          # vxlan/SDN/internal
          #rhel.vm.network :private_network, ip: "192.168.50.2", :netmask => "255.255.255.0", auto_config: false
          rhel.vm.provision :shell, :path => "prepare_rhel.sh"
          rhel.vm.provision :shell, :path => "packstack_it.sh"
          rhel.vm.provision :shell, :path => "setup_os_networks.sh"
          rhel.vm.provider "virtualbox" do |vb|
            vb.name   = "osaio"
            vb.memory = "8192"
            vb.cpus = "4"
          end
  end
end
EOF
```


# Create install scripts

Here we create three scripts, first one to fix RH repos, second one to setup and run packstack to install OpenStack, and third one to setup public network for OpenStack.

```
cat >prepare_rhel.sh<<EOF
# https://access.redhat.com/documentation/en/red-hat-enterprise-linux-openstack-platform/version-7/installation-reference/
#

subscription-manager register --username YOUR_RHN_USERNAME --password 'YOUR_RHN_PASSWD'
subscription-manager attach --pool 'YOUR_RHN_POOL'
subscription-manager repos --disable=*
subscription-manager repos --enable=rhel-7-server-rpms \
        --enable=rhel-7-server-optional-rpms \
        --enable=rhel-7-server-rh-common-rpms \
        --enable=rhel-7-server-extras-rpms
yum install -y yum-plugin-priorities wget yum-utils vim
sudo yum update -y
systemctl stop NetworkManager
systemctl disable NetworkManager
EOF
```

```
cat >packstack_it.sh<<EOF
# https://access.redhat.com/documentation/en/red-hat-enterprise-linux-openstack-platform/version-7/installation-reference/
#
cat > /etc/yum.repos.d/RH7-RHOS-8.0.repo<<EOF
[RH7-RHOS-8.0]
name=RH7-RHOS-8.0
baseurl=ftp://partners.redhat.com/this_part_messed_up/OpenStack/8.0/OSP-8-Beta-5/RH7-RHOS-8.0/x86_64/os/
gpgcheck=0
enabled=1
priority = 1
keepcache = 0
EOF

packstack --install-hosts=192.168.56.3 \
        --os-controller-host=192.168.56.3 \
        --novacompute-privif=lo \
        --os-neutron-l3-ext-bridge=provider \
        --os-neutron-ml2-type-drivers=vxlan,flat \
        --os-neutron-l2-agent=openvswitch \
        --os-neutron-ovs-bridge-mappings=physnet-external:br-ex \
        --os-neutron-ovs-bridge-interfaces=br-ex:eth2 \
        --os-neutron-ovs-tunnel-if=eth2 \
        --provision-demo=n \
        --os-swift-install=y \
        --nagios-install=n \
        --default-password=osdemo \
        --os-heat-install=y \
        --os-client-install=y \
        --os-trove-install=y \
        --os-trove-install=y \
        --os-neutron-lbaas-install=y \
        --neutron-fwaas=y \
        --os-neutron-vpnaas-install=y \
        --os-manila-install=y
EOF
```

```
cat > setup_os_networks.sh <<EOF
# this script nicked from here: http://captainkvm.com/2015/04/packstack-for-rhel-osp-revisited/#more-818

PUB_NAME="Public"
PUB_SUB_NAME="Internet"
PUB_NW="192.168.30.0/24"
PUB_GW=192.168.30.1
PUB_START=192.168.30.60
PUB_END=192.168.30.100
PRIV_NAME="Private"
PRIV_SUB_NAME="dmz1"
PRIV_NW="192.168.100.0/24"
PRIV_GW=192.168.100.1
PRIV_START=192.168.100.20
PRIV_END=192.168.100.100
RTR_NAME=Router_dmz1_pub
PHYS_NET=physnet-external

# sourch admin
source /root/keystonerc_admin
ADMIN_TENANT_ID=$(openstack user show -f shell admin|grep ^id|cut -d'"' -f2)

# create the public network and subnet (FLAT)
neutron net-create $PUB_NAME --tenant-id $ADMIN_TENANT_ID --provider:network_type flat \
  --provider:physical_network $PHYS_NET --router:external=True --shared

neutron subnet-create $PUB_NAME $PUB_NW --name $PUB_SUB_NAME --enable_dhcp=False \
  --allocation-pool start=$PUB_START,end=$PUB_END --gateway=$PUB_GW

# create the private network and subnet
neutron net-create $PRIV_NAME
neutron subnet-create $PRIV_NAME $PRIV_NW --name $PRIV_SUB_NAME \
  --allocation-pool start=$PRIV_START,end=$PRIV_END --gateway=$PRIV_GW

# create the router, interface for the private subnet, and gateway for the public subnet
neutron router-create $RTR_NAME
neutron router-interface-add $RTR_NAME $PRIV_SUB_NAME
neutron router-gateway-set $RTR_NAME $PUB_NAME

# restart the OpenStack Neutron services
systemctl restart neutron-l3-agent.service
systemctl restart neutron-server.service
EOF
```

# Run Vagrant to install it all

This step now takes some while, perhaps an hour.
```
vagrant up
```


# Login to OpenStack
Browse to address http://192.168.56.3/ and login as admin : osdemo.
