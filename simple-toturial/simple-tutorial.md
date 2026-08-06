# OpenStack All-in-One (AIO) Deployment Guide  (ZERO TO HERO)

This guide details the deployment of a functional OpenStack environment using **Kolla-Ansible** on a single Ubuntu 24.04 node. This environment is designed for laboratory use, functional testing, and educational purposes.

## Architecture & Infrastructure
This AIO deployment assumes a virtualized host provisioned within a **vCenter** environment. I recommend using **Terraform** or **Ansible** for the initial provisioning of the virtual machine to ensure consistency and repeatability.

### Node Prerequisites
Before initiating the OpenStack deployment, ensure the target Ubuntu 24.04 node is configured with:
- **Hostname:** Proper FQDN configuration.
- **Networking:** Static IP configuration and correct DNS resolution.
- **SSH:** Passwordless SSH access.
- **Docker:** Latest stable version installed and configured.
- **Time/Date:** NTP/Chronyd configured to prevent clock skew across services.

---
**impotant note:**

*Its importnat to use local proxy repository to use for apt - docker image - pypi ,... ex: setting up nexus repositiory
Also make sure you have terraform node to implement vm to openstack with terraform (IAC)
*******



###### Hardware:

Make sure virtualization is enabled in your host’s BIOS.

Enough resources on the host to run the VMs. in my case:

16 G ram,
1*8 cpu,
120G HDD


We will use Cinder and NFS to store VM images.

2x physical NICs are needed. Their configuration is in /etc/netplan/50-cloud-init.yaml Here:

ens33 is the primary NIC, with IP 191.168.204.92

Make sure to have dhcp6: false in the netplan for that section.

ens37 is the secondary NIC, which should not have an IP assigned.

Disable its DHCP, set dhcp4: false and dhcp6: false for ens37

To apply changes to the configuration file: sudo netplan apply



With a sudo-capable kaosu user for our OpenStack Kolla Ansible installation: 

sudo adduser kaosu; sudo usermod -aG sudo kaosu


a /openstack directory for installing the different components: 

sudo mkdir /openstack

Networking with routing capabilities (i.e., a home router connecting to the internet). For our private network:
The vmware gateway is 192.168.204.2

We will use a static IP for the primary NIC (here ens33 on 192.168.204.92)

We reserved a range of IPs on the subnet that are unused, consecutive, and not assigned to the router’s DHCP range. We will use an IP range of 44 IPs: 192.168.204.155 – 192.168.204.199

We reserved one unused IP for the OpenStack connection; here 192.168.204.254.

## Deployment and initialization Workflow
The deployment process is divided into 5 distinct phases to ensure a clean, reproducible installation:

### Phase 1: Pre-Deployment
In this phase, we ensure the host OS is prepared to handle the virtualization requirements. We install the Hardware Enablement (HWE) kernel to ensure optimal driver support and performance.
```bash
sudo su
apt update && apt upgrade -y
apt install -y linux-generic-hwe-24.04
reboot
```


###### Passwordless sudo
To make our kaosu user use the sudo command without being prompted for a password:

sudo visudo -f /etc/sudoers.d/kaosu-Overrides

```bash
# Add and adapt kaosu as needed
kaosu ALL=(ALL) NOPASSWD:ALL

# save the file and test in a new terminal or login
sudo echo works
```

###### NFS for Cinder
Additional details available here and here.

We want to use NFS on /openstack/nfs to store Cinder-created volumes:

```bash 
# Install nfs server
sudo apt-get install -y nfs-kernel-server

# Create the destination directory and make it nfs-permissions ready
sudo mkdir -p /openstack/nfs
sudo chown nobody:nogroup /openstack/nfs

# edit the `exports` configuration file
sudo nano /etc/exports

# Wihin this file: add the directory and the access host to the authorized list
/openstack/nfs       192.168.204.92(rw,sync,no_subtree_check)

# After saving, restart the nfs server
sudo systemctl restart nfs-kernel-server

# Prepare the cinder configuration to enable the NFS mount
sudo mkdir -p /etc/kolla/config
sudo nano /etc/kolla/config/nfs_shares

# Add the "remote" to mount in the file and save
192.168.204.92:/openstack/nfs
```

Kolla Ansible OpenStack (KAOS)
Latest instructions available here.

We will work from/openstack/kaos for this install as the kaosu user (we recommend the use of a tmux).

Preparation


``` 
cd /openstack
sudo mkdir kaos
sudo chown $USER:$USER kaos
cd kaos

# Install a few things that might otherwise fail during ansible prechecks
sudo apt-get install -y git python3-dev libffi-dev gcc \
  libssl-dev build-essential libdbus-glib-1-dev libpython3-dev \
  cmake libglib2.0-dev python3-venv python3-pip

# Activate a venv
python3 -m venv venv
source venv/bin/activate
pip install -U pip

# Install extra python packages
pip install docker pkgconfig dbus-python

# Install Kolla Ansible from git


pip install --no-cache-dir "git+https://opendev.org/openstack/kolla-ansible@stable/2026.1#egg=kolla-ansible"

pip show kolla-ansible

# Create the /etc/kolla director, and populate it
sudo mkdir -p /etc/kolla
sudo chown $USER:$USER /etc/kolla
cp -r venv/share/kolla-ansible/etc_examples/kolla/* /etc/kolla
# we are going to do an all-in-one (single host) install, copy it in the current folder for easy edits
cp venv/share/kolla-ansible/ansible/inventory/all-in-one .

# Install Ansible Galaxy requirements
kolla-ansible install-deps

# generate random passwords (stored into /etc/kolla/passwords.yml)
kolla-genpwd
```
Edit and adapt the globals file as follows (search for matching keys):

```
sudo nano /etc/kolla/globals.yml


kolla_base_distro: "ubuntu"
kolla_internal_vip_address: "192.168.204.254"
network_interface: "ens33"
neutron_external_interface: "ens37"
enable_cinder: "yes"
enable_cinder_backend_nfs: "yes"
docker_registry: "quay-repo.local"

```
also Before we try the deployment, let’s ensure the Python interpreter is the venv one: at the top of the /openstack/kaos/all-in-one file, add:

```

nano /openstack/kaos/all-in-one

# add to first line of file 
localhost ansible_python_interpreter=/openstack/kaos/venv/bin/python
```

### Phase 2: Deployment
Installation and configuration of OpenStack services using **Kolla-Ansible**. We define the environment parameters via `globals.yml` and the inventory files.

- **Inventory Configuration:** Setting up the `all-in-one` inventory file.
- **Globals setup:** Configuring essential parameters such as network interfaces, image source, and storage backend.
- **Kolla-Ansible bootstrap:** Running pre-deployment checks and pulling container images.
- **Deployment:** Executing the final deployment command using `kolla-ansible deploy`.


As the kaosu user in /openstack/kaos with the venv activated:
```
cd /openstack/kaos

source venv/bin/python/activate || source venv/bin/activate
```
Bootstrap the host:
```
kolla-ansible bootstrap-servers -i ./all-in-one
```
Do pre-deployment checks for the host:
```
kolla-ansible prechecks -i ./all-in-one
```
pull all required images
```
kolla-ansible pull -i ./all-in-one
```

Perform the OpenStack deployment:
```
kolla-ansible deploy -i ./all-in-one
```

If all goes well, you will have a PLAY RECAP at the end of a successful install, which might look similar to the following:

``` 
PLAY RECAP ****...
localhost                  : ok=425  changed=280  unreachable=0    failed=0    skipped=249  rescued=0    ignored=1

``` 

after while:
curl http://192.168.204.254

open in browser:

http://192.168.204.254

Username: admin

password: from 
```
sudo grep '^keystone_admin_password:' /etc/kolla/passwords.yml
```


### Phase 3: Post-Deployment
Once the containers are successfully deployed, we finalize the environment setup to make it functional.

- Generating OpenStack administrative credentials (`admin-openrc.sh`)
- Initializing the OpenStack CLI environment
- Verifying container health and service responsiveness


(still using the venv)

OpenStack command line
Install the python openstack command:
```
pip install python-openstackclient -c https://releases.openstack.org/constraints/upper/master
```
OpenStack configuration file
Create multiple post-deployment scripts, including the admin-openrc.sh and cloud.yml files:
```
kolla-ansible post-deploy -i ./all-in-one
That file should be added to your default config:

```
Cloud Init: Run once
(requires the venv, the openstack command line, the cloud.yml file, and the generated/etc/kolla/admin-openrc.sh script)

In /openstack/kaos, there is a venv/share/kolla-ansible/init-runonce script to create some of the basic configurations for your cloud. Most end users will modify their EXT_NET_CIDR, EXT_NET_RANGE, and EXT_NET_GATEWAY variables.

```
nano venv/share/kolla-ansible/init-runonce

EXT_NET_CIDR=192.168.204.0/24

EXT_NET_RANGE=
start=192.168.204.155,end=192.168.204.199

EXT_NET_GATEWAY=192.168.204.2

#####after save run it###########

./venv/share/kolla-ansible/init-runonce

```
The proposed my-init-runonce.sh executable (ie chmod +x it) script uses larger tiny images (5GB, as a Ubuntu server is over 2GB), and other instances only use a base image of 20GB (since you can specify your preferred disk image size during the instance creation process), its instance names following the m<number_of_cores> naming convention and adds xxlarge and xxxlarge memory instances.

Adapt the USER CONF section based on your system and preferences.
```
% ./my-init-runonce.sh
[...]
-- Attempt to add external-net (if not already present)
[...]
-- Attempt to configure Security Groups: ssh and ICMP (ping)
[...]
-- Attempt to create and add a default id_ecdsa key to nova (if not already present)
[...]
-- Setting quota defaults following user values
[...]
-- Creating defaults flavors (instance type) (if not already present)
[...]
Done
```
Once run, we should have:

An external-net: the pool from which your floating IPs will be obtained.
Added ssh and ICMP to the admin project’s default security group.
Created a default ssh key (mykey) and added it to the admin user.
Set the admin’s project default quotas (this will not propagate to other projects, but the CLI logic can with the right project_id).
```
Created a list of default flavors, such as:
% source /etc/kolla/admin-openrc.sh
% openstack flavor list
+----+--------------+-------+------+-----------+-------+-----------+
| ID | Name         |   RAM | Disk | Ephemeral | VCPUs | Is Public |
+----+--------------+-------+------+-----------+-------+-----------+
| 1  | m1.tiny      |   512 |    5 |         0 |     1 | True      |
| 2  | m2.tiny      |   512 |    5 |         0 |     2 | True      |
| 3  | m2.small     |  2048 |   20 |         0 |     2 | True      |
| 4  | m2.medium    |  4096 |   20 |         0 |     2 | True      |
| 5  | m4.large     |  8192 |   20 |         0 |     4 | True      |
| 6  | m8.xlarge    | 16384 |   20 |         0 |     8 | True      |
| 7  | m16.xxlarge  | 32768 |   20 |         0 |    16 | True      |
| 8  | m32.xxxlarge | 65536 |   20 |         0 |    32 | True      |
+----+--------------+-------+------+-----------+-------+-----------+
FYSA: From the UI, it is possible to add new flavors from Admin -> Compute -> Flavors
```
Post-Installation
Note: kolla-ansible or openstack requires the venv to be activated and source /etc/kolla/admin-openrc.sh to be performed for the commands to have the correct configuration information. As kaosu:
```
cd /openstack/kaos
source /etc/kolla/admin-openrc.sh
source venv/bin/activate
```


New admin user (UI)
Login to your OpenStack instance by going to the web dashboard (horizon, available on port 80) at http://192.168.204.254

The default admin user’s password can be obtained using:

fgrep keystone_admin_password /etc/kolla/passwords.yml
Using Project -> Compute -> Overview gives you a list of used and available resources.

Create a new project and another admin user for your account. As the admin user:

In the Identity -> Projects (left column), Create Project and choose a name. For this example, we will use newprojectname. That new project does not inherit the existing one’s default values. We will update the quotas in the next section.
In the Identity -> Users (left column), Create User. Provide its User Name and Password (Confirm), assign that user the Primary Project created above, and give it the Admin Role. Enable the account. For this example, we will use newadminuser.



### Phase 4: Cluster Preparation
Setting the foundation for running virtual machines. This phase covers:

- **Networking:** Configuring external/internal networks, subnets, and L3 routers
- **Compute:** Defining flavor profiles (vCPU, RAM, Disk)
- **Security:** Configuring security groups, rules, and SSH keypairs
- **Images:** Uploading base cloud images such as Ubuntu or CirrOS


```
openstack project create my-first-project
openstack project create my-first-project
openstack role add --project my-first-project --user myuser member
openstack keypair create --public-key ~/.ssh/id_ecdsa.pub mykey --user myuser


# Adapt newprojectname
MY_PROJECT_ID=$(openstack project list | awk '/ newprojectname / {print $2}')
MY_SEC_GROUP=$(openstack security group list --project ${MY_PROJECT_ID} | awk '/ default / {print $2}')
# check values are assigned
echo $MY_PROJECT_ID
echo $MY_SEC_GROUP

openstack security group rule create --ingress --ethertype IPv4 --protocol icmp ${MY_SEC_GROUP}
openstack security group rule create --ingress --ethertype IPv4 --protocol tcp --dst-port 22 ${MY_SEC_GROUP}

openstack quota set --force --instances 10 ${MY_PROJECT_ID}
openstack quota set --force --cores 32 ${MY_PROJECT_ID}
openstack quota set --force --ram 96000 ${MY_PROJECT_ID}
openstack quota set --force --floating-ips 10 ${MY_PROJECT_ID}
```

###### Add an Ubuntu image to Glance
Go to https://cloud-images.ubuntu.com/ and select the distro you want (here, we will use Noble Numbat/Ubuntu 24.04’s most current image). Copy the URL of the QCow2 UEFI/GPT Bootable disk image of your choice.
```
cd /openstack
sudo mkdir cloudimg
sudo chown $USER:$USER cloudimg
cd cloudimg

# Name it with the OS information and the date shown in the "Last modified" column
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img -O ubuntu2404-20250403.img
```

Use the openstack command line to add the image to the list of available images for all users of our cloud OS, giving it a name that indicates its content:
```
openstack image create --disk-format qcow2 --container-format bare --public --property os_type=linux --file ubuntu2404-20250403.img ubuntu2404server-20250403

```
Once completed, a table with details of the new image added to our OpenStack installation will appear. 

From our new admin user’s UI, select Project -> Compute -> Images, and we will see the added image listed.

### Phase 5: Creating the First Instance
The final validation step. We launch a virtual machine (instance) to verify end-to-end connectivity, image booting, and network flow.

**Success Criteria:** The instance boots successfully, passes cloud-init, and is accessible via SSH.


```
nano cloud-config.yml

#cloud-config

hostname: ubuntu01

users:
  - default

ssh_pwauth: true

chpasswd:
  list: |
    ubuntu:OpenStack123
  expire: false

package_update: true

packages:
  - qemu-guest-agent
  - curl
  - htop

runcmd:
  - systemctl enable qemu-guest-agent
  - systemctl start qemu-guest-agent

```
this is your costumise vm preparation. other ways is create your golden image vm.

to create vmwith openstack client run:

```

openstack server create \
  --flavor m1.small \
  --image ubuntu2404server-20250403 \
  --network myproject-net \
  --security-group your_unique_security_group_name_or_id \
  --key-name mykey2 \
  --user-data cloud-init.yaml \
  ubuntu01


openstack image list
openstack network list
openstack security group list
openstack keypair list


ssh -i /path/to/your/mykey ubuntu@192.168.204.100
```
ls -alh /openstack/nfs will show the file for our newly created disk volume.



#### Network and Router setup in neutron

From our new admin user’s UI (which should start in our recently added project), select 

Project -> Network -> Network Topology. 

This should show a graph with only the external-net.

We need a network and a router added for VMs to communicate.

*Network*

Select Create Network

*Network tab:*

Name it: we recommend project-net with project reflecting our project’s name.

Check Enable Admin State to make sure it is active.

Uncheck Shared, this network is only for this project.

Check Create Subnet; we need to configure the IP details for this subnet.

There is no need to modify Availability Zone Hints or MTU

Click Next.

*Subnet tab:*

Name it: a similar project-subnet.

For the Network Address, use a private IP range not currently used in our network, such as 192.168.204.0/24; 

subnets must be independent and not currently in use.

Select IPv4.

Use 192.168.204.2 for the Gateway IP; it must be in the same IP range as your subnet.

Uncheck Disable Gateway.

Click Next.

*Subnet Details tab:*

Check Enable DHCP. We want our VM instances to get IPs automatically when they start.

For Allocation Pool use something unused within the subnet range, for example, 192.168.204.155,192.168.204.199.

DNS Name Servers (one entry per line) use Google (8.8.8.8, 8.8.4.4) or CloudFlare (1.1.1.1).

No need to add any Host Routes.

Click Create.

You now have a new network ready to be used with VMs. We still need a router.

*Router*

Select Create Router:

Name it: project-router.

Check Enable Admin State to make sure it will be active.

Select the external-net External Network.

Check Enable SNAT since we do have an external network.

Leave Availabilty Zone Hints as is.

We now have a router connected to the external network. 

The IP for the router on the external network is automatically selected from the pool.


The router has yet to be connected to the “project network.” Hover over the “router” and select Add Interface. Select the project-subnet Subnet and leave the IP Address unspecified; it will use the configured gateway.

When we return to the Network Topology page, we will see an external-net connected to our project-net by our project-router.

The instance is designed to be remotely accessed using SSH. We need to assign our instance a “Floating IP”: a public IP address that can be dynamically associated with a private instance, allowing it to be accessible from outside the private cloud.

**Floating IPs****

With our instance Running, its IP Address is within our project’s subnet range.

We need to obtain a Floating IP to access the instance via SSH.

In the Actions (right) submenu for our instance row, select Associate 

*Floating IP:*

None are listed; click the + and Allocate IP from our external-net pool.

An IP will now show in the IP Address dropdown. Make sure the Port to be associated matches our u24test instance and Associatethem.
The IP Address column will now show two IPs: one from the project-subnet DHCP range and one from the external-net pool.

From your kaosu user, we can ssh into the host’s created floating IP using the authorized ssh key and the default cloud image user of ubuntu.

For example:

``` 
# Adapt the IP to match your floating IP
ssh -i ~/.ssh/id_ecdsa ubuntu@192.168.204.190
[...]
Welcome to Ubuntu 24.04.2 LTS (GNU/Linux 6.8.0-57-generic x86_64)
[...]
ubuntu@u24test:~$
```

From there, you can confirm that your instance can connect to the Internet by running sudo apt update && sudo apt -y upgrade.

Securely accessing Horizon using a reverse proxy
If you have a reverse proxy setup on another host and want to benefit from https on horizon (the dashboard):

In your reverse proxy, configure the Proxy Host as you would typically; here, we will use os.example.com
Run:

```
sudo nano /etc/kolla/horizon/_9999-custom-settings.py

###and add to it:
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")
CSRF_TRUSTED_ORIGINS = [ 'https://os.example.com' ]
```

Restart horizon using docker kill horizon. Wait a few seconds, and your access via https://os.example.com should be functional
ie, not present us with a csrf_failure=Origin checking failed - https%3A//os.example.com does not match any trusted origin error (in the address bar).
FYSA, your installer has named all the containers using the name of the service they provide, so horizon is one of them


###### Implement vm via terraform

in terraform node add a working directory for your project:
```
su -
mkdir tr-os
cd tr-os/
nano admin-openrc.sh
chmod +x admin-openrc.sh
source ~/tr-os/admin-openrc.sh

nano providers.tf

terraform {
  required_version = ">= 1.0.0"
  required_providers {
    openstack = {
      source  = "terraform-provider-openstack/openstack"
      version = "~> 2.1.0" 
    }
  }
}
provider "openstack" {}

nano terraform.tfvars

image_name            = "ubuntu26-20260720"
flavor_name           = "m1.small"
internal_network_name = "demo-net"
external_network_name = "public"
region_name           = "RegionOne"
keypair_name          = "zizi"
public_key_file       = "~/.ssh/id_ed25519.pub"
vm_name               = "ubuntu-26-terraform-demo"
admin_password        = "123"



nano main.tf

terraform {
  required_version = ">= 1.3.0"

  required_providers {
    openstack = {
      source  = "terraform-provider-openstack/openstack"
      version = "2.1.0"
    }
  }
}

provider "openstack" {
  # Credentials از محیط (admin-openrc.sh) خوانده می‌شود.
}

###############################################################################
# Variables
###############################################################################

variable "region_name" {
  description = "OpenStack region"
  type        = string
  default     = "RegionOne"
}

variable "image_name" {
  description = "Image name in OpenStack"
  type        = string
  default     = "ubuntu26-20260720"
}

variable "flavor_name" {
  description = "Flavor name"
  type        = string
  default     = "m1.small"
}

variable "internal_network_name" {
  description = "Tenant internal network name"
  type        = string
  default     = "demo-net"
}

variable "external_network_name" {
  description = "External network name for Floating IP (set in terraform.tfvars)"
  type        = string
}

variable "keypair_name" {
  description = "Keypair name"
  type        = string
  default     = "zizi"
}

variable "public_key_file" {
  description = "Path to SSH public key"
  type        = string
  default     = "~/.ssh/id_ed25519.pub"
}

variable "vm_name" {
  description = "VM name"
  type        = string
  default     = "ubuntu-26-terraform-demo"
}

variable "admin_password" {
  description = "Password for default ubuntu user (cloud-init)."
  type        = string
  sensitive   = true
  default     = "123"
}

###############################################################################
# Data sources
###############################################################################

data "openstack_images_image_v2" "ubuntu" {
  name        = var.image_name
  most_recent = true
}

data "openstack_compute_flavor_v2" "vm_flavor" {
  name = var.flavor_name
}

data "openstack_networking_network_v2" "internal_network" {
  name = var.internal_network_name
}

data "openstack_networking_network_v2" "external_network" {
  name     = var.external_network_name
  external = true
}

###############################################################################
# Keypair
###############################################################################

resource "openstack_compute_keypair_v2" "my_keypair" {
  name       = var.keypair_name
  public_key = file(pathexpand(var.public_key_file))
}

###############################################################################
# Security Group + Rules
###############################################################################

resource "openstack_networking_secgroup_v2" "ubuntu_secgroup" {
  name        = "${var.vm_name}-secgroup"
  description = "Security group for ${var.vm_name}"
}

resource "openstack_networking_secgroup_rule_v2" "ssh" {
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "tcp"
  port_range_min    = 22
  port_range_max    = 22
  remote_ip_prefix  = "0.0.0.0/0"
  security_group_id = openstack_networking_secgroup_v2.ubuntu_secgroup.id
}

resource "openstack_networking_secgroup_rule_v2" "icmp" {
  direction         = "ingress"
  ethertype         = "IPv4"
  protocol          = "icmp"
  remote_ip_prefix  = "0.0.0.0/0"
  security_group_id = openstack_networking_secgroup_v2.ubuntu_secgroup.id
}

resource "openstack_networking_secgroup_rule_v2" "egress_ipv4" {
  direction         = "egress"
  ethertype         = "IPv4"
  remote_ip_prefix  = "0.0.0.0/0"
  security_group_id = openstack_networking_secgroup_v2.ubuntu_secgroup.id
}

###############################################################################
# cloud-init (enable password login)
###############################################################################

locals {
  cloud_init = <<-CLOUD_CONFIG
    #cloud-config
    hostname: ${var.vm_name}
    manage_etc_hosts: true

    users:
      - default

    ssh_pwauth: true

    chpasswd:
      expire: false
      list:
        - ubuntu:${var.admin_password}

    runcmd:
      - systemctl enable ssh || true
      - systemctl restart ssh || true
  CLOUD_CONFIG
}

###############################################################################
# Compute instance
###############################################################################

resource "openstack_compute_instance_v2" "ubuntu_vm" {
  name            = var.vm_name
  image_id        = data.openstack_images_image_v2.ubuntu.id
  flavor_id       = data.openstack_compute_flavor_v2.vm_flavor.id
  key_pair        = openstack_compute_keypair_v2.my_keypair.name
  security_groups = [openstack_networking_secgroup_v2.ubuntu_secgroup.name]
  region          = var.region_name

  user_data = local.cloud_init

  network {
    uuid = data.openstack_networking_network_v2.internal_network.id
  }
}

###############################################################################
# Floating IP + Associate
###############################################################################

resource "openstack_networking_floatingip_v2" "ubuntu_vm_fip" {
  pool   = data.openstack_networking_network_v2.external_network.name
  region = var.region_name
}

resource "openstack_compute_floatingip_associate_v2" "ubuntu_vm_fip_association" {
  floating_ip = openstack_networking_floatingip_v2.ubuntu_vm_fip.address
  instance_id = openstack_compute_instance_v2.ubuntu_vm.id
  region      = var.region_name
}

###############################################################################
# Outputs
###############################################################################

output "floating_ip" {
  description = "Floating IP"
  value       = openstack_networking_floatingip_v2.ubuntu_vm_fip.address
}

output "ssh_command" {
  description = "SSH command"
  value       = "ssh ubuntu@${openstack_networking_floatingip_v2.ubuntu_vm_fip.address}"
}

output "password" {
  description = "Ubuntu user password"
  value       = var.admin_password
  sensitive   = true
}




```

on openstack node check this:
```
(venv) root@os:/openstack/kaos# openstack token issue
+------------+-------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                   |
+------------+-------------------------------------------------------------------------------------------------------------------------+
| expires    | 2026-08-07T10:43:22+0000                                                                                                |
| id         | gAAAAABqdGVKDXjsq-jtzmgcvLfLTYOMz_XQ7uf8PShHRg2rnYHtWHgj4vcnGftV2UYqzTGRVcLs_2C_QmTaImEPw_86qN0DOFudq_hGiqeyatEjbbzVzS- |
|            | t5VVuECLsmbAD9z7f38KL-jabU6CzoB5zbW6CO4Z8ay14UandIixbeUJEe6jVcgw                                                        |
| project_id | 6ca11d405f404fb09ba4ec14f730b1e2                                                                                        |
| user_id    | 669e59975fcc46fbadfa77d672492300                                                                                        |
+------------+-------------------------------------------------------------------------------------------------------------------------+
(venv) root@os:/openstack/kaos# 

```
on terraform node :
```
source ~/tr-os/admin-openrc.sh
terraform fmt
terraform init
terraform validate
terraform providers
terraform plan
terraform plan -out=tfplan
terraform apply tfplan


terraform show tfplan
terraform apply tfplan
```
after apply successed you can see output:
```
root@utilsrv:~/tr-os# terraform apply tfplan-new
openstack_compute_keypair_v2.my_keypair: Creating...
openstack_networking_secgroup_v2.secgroup_ssh: Creating...
openstack_networking_secgroup_v2.secgroup_ssh: Creation complete after 0s [id=21015aa0-189f-4726-b958-3c272c6b95dd]
openstack_networking_secgroup_rule_v2.rule_ssh: Creating...
openstack_networking_secgroup_rule_v2.rule_icmp: Creating...
openstack_compute_keypair_v2.my_keypair: Creation complete after 1s [id=zizi]
openstack_compute_instance_v2.ubuntu_vm: Creating...
openstack_networking_secgroup_rule_v2.rule_icmp: Creation complete after 1s [id=3d1b4d8d-8932-4986-9ef8-9c974ae661f5]
openstack_networking_secgroup_rule_v2.rule_ssh: Creation complete after 1s [id=aaa2cf2e-9b51-4901-b676-b388fa080d35]
openstack_compute_instance_v2.ubuntu_vm: Still creating... [00m10s elapsed]
openstack_compute_instance_v2.ubuntu_vm: Still creating... [00m20s elapsed]
openstack_compute_instance_v2.ubuntu_vm: Still creating... [00m30s elapsed]
openstack_compute_instance_v2.ubuntu_vm: Still creating... [00m40s elapsed]
openstack_compute_instance_v2.ubuntu_vm: Still creating... [00m50s elapsed]
openstack_compute_instance_v2.ubuntu_vm: Still creating... [01m00s elapsed]
openstack_compute_instance_v2.ubuntu_vm: Still creating... [01m10s elapsed]
openstack_compute_instance_v2.ubuntu_vm: Still creating... [01m20s elapsed]
openstack_compute_instance_v2.ubuntu_vm: Still creating... [01m30s elapsed]
openstack_compute_instance_v2.ubuntu_vm: Still creating... [01m40s elapsed]
openstack_compute_instance_v2.ubuntu_vm: Creation complete after 1m45s [id=445f4074-6f60-42e9-a8ff-67cd1292ffbd]

Apply complete! Resources: 5 added, 0 changed, 0 destroyed.

Outputs:

instance_ip = "10.0.0.219"



terraform output -raw floating_ip
terraform output -raw ssh_command
terraform output -raw password

```

on openstack node check instance:
```
(venv) root@os:/openstack/kaos# openstack server list --name ubuntu-26-terraform-demo
+--------------------------------------+--------------------------+--------+---------------------+-------------------+----------+
| ID                                   | Name                     | Status | Networks            | Image             | Flavor   |
+--------------------------------------+--------------------------+--------+---------------------+-------------------+----------+
| 445f4074-6f60-42e9-a8ff-67cd1292ffbd | ubuntu-26-terraform-demo | ACTIVE | demo-net=10.0.0.219 | ubuntu26-20260720 | m1.small |
+--------------------------------------+--------------------------+--------+---------------------+-------------------+----------+
(venv) root@os:/openstack/kaos# 
```







## Troubleshooting

“reconfigure” if you need to modify globals.yml
If you modify a globals.yml configuration option,
```
cd /openstack/kaos
source venv/bin/activate
kolla-ansible reconfigure -i ./all-in-one
```

More kolla-ansible CLI options at https://docs.openstack.org/kolla-ansible/latest/user/operating-kolla.html.

Broken after a Reboot?
I experienced this in a previous installation. Luckily, it is just a matter of re-running the reconfigure step to make it functional again.

Login as the kaosu user
```
cd /openstack/kaos
source venv/bin/activate

pip3 install -U pip

kolla-ansible -i ./all-in-one --yes-i-really-really-mean-it stop
kolla-ansible -i ./all-in-one install-deps
kolla-ansible -i ./all-in-one prechecks
kolla-ansible -i ./all-in-one reconfigure

sudo docker ps -a
```



refrence:

https://superuser.openinfra.org/articles/kolla-ansible-openstack-installation-ubuntu-24-04/

https://docs.openstack.org/kolla-ansible/latest/user/quickstart.html

https://docs.openstack.org/project-deploy-guide/kolla-ansible/2026.1/quickstart.html#host-machine-requirements

