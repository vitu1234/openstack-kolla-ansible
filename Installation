This guide provide step by step  instructions to deploy Openstack using Kolla Ansible on bare metal servers (Debian or Ubuntu)
# Host machine requirements
The host machine must satisfy the following minumum requirements:
- 2 network interfaces
- 8GB main memory
- 40GB disk space

# Topology server (example)
1 control node hosted control services and ansible role: 192.168.24.13
2 compute node: 192.168.24.14, 192.168.24.15

Connect all host under ssh without password, using the same user in each node
On control node
```
ssh-keygen -t rsa
```
On (all) compute nodes
```
scp root@192.168.24.13:~/.ssh/id_rsa.pub ./
cat id_rsa.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```


# Setup on Control node
## Install dependenies for the virtual environment
Typically commands that use the system package manager in this section must be run with root privileges.

1. Install the virtual environment dependencies.
```
sudo apt update
sudo apt upgrade
sudo apt install python3-dev python3-venv libffi-dev gcc libssl-dev git
sudo apt install python3-venv
```
2. Create a virtual environment and activate it.

```
python3 -m venv $HOME/kolla-openstack
source $HOME/kolla-openstack/bin/activate
```
3. Ensure the latest version of pip is installed:
```
pip install -U pip
```
4. Install Ansible.
```
# pip install 'ansible-core>=2.13,<=2.14.2'
pip install 'ansible-core>=2.15,<2.16.99'

pip install 'ansible>=6,<8'
```

## Install kolla-ansible
1. create an ansible configuration file on your home directory
```
vi $HOME/ansible.cfg
```
```
[defaults] 
host_key_checking=False 
pipelining=True 
forks=100
```

2. Install kolla-ansible and its dependencies using pip
latest version
```
pip install git+https://opendev.org/openstack/kolla-ansible@master
```
custom to openstack-release (example - "yoga")
```
pip install git+https://opendev.org/openstack/kolla-ansible@stable/yoga
```
3. Create the /etc/kolla directory.
```
sudo mkdir /etc/kolla
sudo chown $USER:$USER /etc/kolla
```
4. Copy globals.yaml and passwords.yaml to /etc/kolla directory, then Copy multinode inventory file to the current directory.
```
cp $HOME/kolla-openstack/share/kolla-ansible/etc_examples/kolla/* /etc/kolla/
cp $HOME/kolla-openstack/share/kolla-ansible/ansible/inventory/multinode .
```
## Install Ansible Galaxy requirements
```
kolla-ansible install-deps
```
## Prepare initial configuration
1. Kolla passwords
Passwords used in our deployment are stored in /etc/kolla/passwords.yml file. All passwords are blank in this file and have to be filled either manually or by running random password generator

```
kolla-genpwd
```
2. Kolla globals.yml
globals.yml is the main configuration file for Kolla Ansible and per default stored in /etc/kolla/globals.yml file. There are a few options that are required to deploy Kolla Ansible:
Edit globals file
```
vi /etc/kolla/globals.yml
```

```
kolla_base_distro: "ubuntu"
kolla_install_type: "source"
openstack_release: "yoga"
kolla_internal_vip_address: "192.168.24.13"
network_interface: "eno1"
neutron_external_interface: "eno2"
enable_haproxy: "no"
enable_aodh: "yes"
enable_neutron_provider_networks: "yes"
enable_vitrage: "yes"
nova_compute_virt_type: "qemu"
```


3. Multinode configuration
```
vi multinode
```

```
[control]
# These hostname must be resolvable from your deployment host
localhost

# The above can also be specified as follows:
#control[01:03]     ansible_user=kolla

# The network nodes are where your l3-agent and loadbalancers will run
# This can be the same as a host in the control group
[network]
localhost

[compute]
localhost
192.168.24.14
192.168.24.15

[monitoring]
localhost
[storage]
localhost

[deployment]
localhost       ansible_connection=local

[baremetal:children]
control
network
compute
storage
monitoring
```
## Deployments
After configuration is set, we can proceed to the deployment phase.

1. Run multinode file to check the connection between hosts

```
ansible -i ./multinode all -m ping
```
2. Bootstrap servers with kolla deploy dependencies:
```
kolla-ansible -i ./multinode bootstrap-servers
```
3. Do pre-deployment checks for hosts:
```
kolla-ansible -i ./multinode prechecks
```
4. Finally proceed to actual Openstack deployment
```
kolla-ansible -i ./multinode deploy
```
When this playbook finishes, OpenStack should be up, running and functional! 


## Install OpenStack command line administration tools
```
pip install python-openstackclient python-neutronclient python-glanceclient
```
Generate OpenStack admin user credentials file
```
kolla-ansible post-deploy
source /etc/kolla/admin-openrc.sh
```

Edit the init-runonce script and configure your public network,that you want to connect to the internet via (xx-yy is the range of IP address)
```
vi kolla-openstack/share/kolla-ansible/init-runonce
```
```
ENABLE_EXT_NET=${ENABLE_EXT_NET:-1}
EXT_NET_CIDR=${EXT_NET_CIDR:-'192.168.24.0/24'}
EXT_NET_RANGE=${EXT_NET_RANGE:-'start=192.168.24.xx,end=192.168.24.yy'}
EXT_NET_GATEWAY=${EXT_NET_GATEWAY:-'192.168.24.1'}
```


comment following line 
```
#$KOLLA_OPENSTACK_COMMAND network create demo-net
#$KOLLA_OPENSTACK_COMMAND subnet create --subnet-range 10.0.0.0/24 --network demo-net --gateway 10.0.0.1 --dns-server 8.8.8.8 demo-subnet
#$KOLLA_OPENSTACK_COMMAND router and subnet demo-router demo-subnet

```
Add config defaut network, gateway and router

```

if [[ $ENABLE_EXT_NET -eq 1 ]]; then
   $KOLLA_OPENSTACK_COMMAND network create --external --provider-physical-network physnet1 --provider-network-type flat public1
   $KOLLA_OPENSTACK_COMMAND subnet create --dhcp --allocation-pool ${EXT_NET_RANGE} --network public1 --subnet-range ${EXT_NET_CIRD} --gateway ${EXT_NET_GATEWAY} --dns-nameserver 8.8.8.8 public1-subnet
   $KOLLA_OPENSTACK_COMMAND router set --external-gateway public1 demo-router
fi

```
Init the configuration
```
kolla-openstack/share/kolla-ansible/init-runonce
```

Obtain the admin credentials from the Kolla password file:

```
vi /etc/kolla/passwords.yml
grep keystone_admin_password /etc/kolla/passwords.yml 
```
Example results for keystone_admin_password:

```
keystone_admin_password: vyntMgfyaZnHU..."
```

Access Dashboard through control node: 192.168.24.20 with admin account and keystone_admin_password

## Re-configure
reconfig all services
```
kolla-ansible -i ./multimode reconfigure
```
recofig a specific service

```
kolla-ansible -i ./multimode reconfigure -t <service-name>
Example: (kolla-ansible -i ./multimode reconfigure -t prometheus)
```
## Destroy Openstack
```
kolla-ansible -i ./multinode destroy --yes-i-really-really-mean-it
```
## Completely uninstall docker from Linux
step 1
```
dpkg -l | grep -i docker
```
step 2: check what installed package you have
```
sudo apt-get purge -y docker-engine docker docker.io docker-ce docker-ce-cli docker-compose-plugin && sudo apt-get autoremove -y --purge docker-engine docker docker.io docker-ce docker-compose-plugin
```
step 3: Delete all images, container, and volumes 

```
sudo rm -rf /var/lib/docker /etc/docker
sudo groupdel docker
sudo rm -rf /var/run/docker.sock
```


# cinder and resize volume

1. Use Space from Extended Physical (or Virtual) Disk

extend the LV to use up all the VG’s free space with
```
lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
```

2. check the LV one more time with `lvdisplay` to make sure it has been extended
3. Resize the file system to make the additional 100% space usable
```
resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
```
