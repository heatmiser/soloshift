# soloshift
Ansible automation for deploying a local "all in one" OpenShift 4 cluster

# Overview

While single system installation of OpenShift 4 is also made possible via [Red Hat CodeReady Containers](https://developers.redhat.com/blog/2019/09/05/red-hat-openshift-4-on-your-laptop-introducing-red-hat-codeready-containers/), some users desire a more complete single system installation of OpenShift 4 that more closely resembles a full, multi-system OpenShift 4 cluster deployment, whether that be for laptop installation or single server lab deployments.

# Cluster components

By default, soloshift deploys a 1-1-1 OpenShift 4 cluster stack comprised of a single master node, single infrastructure node, and single worker node.  In addition, a single utility node is deployed that provides DHCP, tftp, DNS, matchbox, and haproxy services, as well as serving as a location for the OpenShift 4 User Provided Infrastructure installation directory. If base system resources can support it, the cluster can be expanded to utilize a 3 master node control plane, multiple infrastructure nodes, as well as additional worker nodes.  Analyzing the available system resources to see if supporting additional nodes is possible is left as an exercise to the end user.

Requirements
------------

A Linux KVM hypervisor OS, preferably RHEL/CentOS/Fedora with a minimum of 16GB RAM. Note that work is in progress to see if installations on Macs can be supported.

Installation
------------

For CentOS hypervisors, skip to step 2.

For Fedora hypervisors, skip to step 3.

1) If you're using RHEL for the base hypervisor OS, then you'll need to register the system and configure the system subscription first. As root, or via sudo:

- Red Hat username and password

`(hypervisor)# subscription-manager register --username="your_user_name" --password="your_user_password"`

> **NOTE**: Leave out the --password switch if you want to enter your password interactively and not record password in shell history.

- ...or Red Hat organization ID and activation key:

`(hypervisor)# subscription-manager register --activationkey="your_key_name" --org="your_org_id#"`

...then:

`(hypervisor)# subscription-manager repos --disable="*"`

`(hypervisor)# subscription-manager repos --enable="rhel-7-server-rpms"`

`(hypervisor)# subscription-manager repos --enable="rhel-7-server-extras-rpms"`

`(hypervisor)# subscription-manager repos --enable="rhel-7-server-ansible-2.8-rpms"`

2) CentOS hypervisors only

`(hypervisor)# yum -y install epel-release`

3) 

<center>

| Fedora hypervisors only | CentOS/RHEL hypervisors only |
|:-:|:-:|
| `(hypervisor)# dnf -y install git ansible` |  `(hypervisor)# yum -y install git ansible` |

</center>

4) Continue...

`(hypervisor)# git clone https://github.com/heatmiser/soloshift.git`

`(hypervisor)# cd soloshift`

Copy `inventory/group_vars/all/default_vars.yaml`
to `inventory/group_vars/all/my_vars.yaml`

Next, edit `inventory/group_vars/all/my_vars.yaml`

SoloShift employs a utility VM to run the OpenShift 4 User Provided Infrastructure installation, as well as to provide base infrastructure services, such as DHCP, tftp, DNS, matchbox, and haproxy.

If utilizing RHEL for the utility VM, choose either pair of:

- organization ID and activation key
  
or

- username and password 

...and leave the other pair undefined.

* `redhat_subscription_org_id`: Subscription organization ID. If using an activation key, this is required. If comprised of all numbers, surround in double-quotes.
* `redhat_subscription_activationkey`: Activation key to use for host registration.

* `redhat_subscription_username`: If not using an activation key, specify Red Hat username. 
* `redhat_subscription_password`: If not using an activation key, specify Red Hat password.
* `redhat_subscription_pool_regex`: If utilizing username/password, supply a regex to match on the desired pool. For example, to match on a subscription pool named Red Hat Enterprise Server, use "^Red Hat Enterprise Server$"

* `ocp_vms_base_image`: rhel-server-7.7-x86_64-kvm.qcow2 - enter name of RHEL KVM Guest image downloaded from https://access.redhat.com/downloads
* `ocp_vms_openshift_release`: 4.1 - OpenShift version to deploy; 4.1, 4.2, or pre-release
* `ocp_vms_openshift_subdomain`: ocp41 - name for top level DNS sub-domain
* `ocp_vms_openshift_rootdomain`: domain.com - base DNS second-level domain
* `ocp_vms_libvirt_images_location`: Using a vm image storage location different than the default?  Define it here.
* `ocp_vms_net_cidr`: 192.168.8.0/24 - internal subnet for cluster to use
* `ocp_vms_master_count`: 1
* `ocp_vms_infra_count`: 1
* `ocp_vms_worker_count`: 1
* `ocp_vms_openshift_pullsecret_file`: pull-secret.txt - download from https://cloud.redhat.com/openshift/install/metal/user-provisioned

By default, SoloShift deploys VMs utilizing sparse backing files.  When creating VMs, if the size of the requested backing file is greater than the total amount of space available in the volume containing the directory defined by `ocp_vms_libvirt_images_location`, the deploy automation will fail. If you would like to implement storage overcommit in order to bypass this limitation, add the following line to `inventory/group_vars/all/my_vars.yaml`:

* `ocp_vms_storage_overcommit: True`

> **NOTE**: Storage overcommit can potentially lead to completely filling the total amount of space available in the volume containing the directory defined by `ocp_vms_libvirt_images_location`. Be sure to keep this in mind and monitor overall volume capacity

Deploy All-in-One OCP4
------------

Place `pull-secret.txt` in the root of the soloshift directory.

Download your VM image of choice (RHEL7 KVM qcow2 guest image, for example) and place it in the directory defined by `ocp_vms_libvirt_images_location`. Then, update `ocp_vms_base_image` with the name of the image.  If you have configured a non-standard VM images directory location, place the VM image there and make sure to update `ocp_vms_libvirt_images_location` to reflect that location.

If you'd like to adjust the number of vcpus, memory, ram, or disk sizes of the various VM nodes, edit
`roles/ocp4-solo-vmprovision/defaults/main.yml` before proceeding.  The default values are as low as you should go for successful installations. A minumum base hypervisor RAM of 16GB is required (laptop installation was one of the original goals for soloshift).

`(hypervisor)# ansible-galaxy install -p ./roles -r requirements.yaml`

`(hypervisor)# ansible-playbook playbooks/00-ocp-hyper.yaml`

`(hypervisor)# ansible-playbook playbooks/01-ocp-vms.yaml`

`(hypervisor)# ansible-playbook playbooks/02-ocp-util.yaml`

`(hypervisor)# ansible-playbook playbooks/03-ocp-init.yaml`

Either access the util vm console via virt-viewer or ssh into the util vm as root. `ocp_vms_password` is the root password, set in the defaults for the `ocp4-solo-vmprovision` role. If you left `ocp_vms_net_cidr` at the default internal subnet to use, then the util node will be at 192.168.8.8.  There will be an SSH key pair in your user's .ssh directory prefixed with whatever was set for `ocp_vms_openshift_subdomain`.  You can use that private key to ssh in to the util node as root.

After logging in to the util node as root, execute:

`(util)# openshift-install --dir=/root/ocp4upi wait-for bootstrap-complete --log-level debug`

Eventually, you'll see a log message saying that it's ok to shutdown the bootstrap machine. Back on the hypervisor system, shutdown the bootstrap node, either via the Virtual Machine Manager or `virsh` command line tool.

Next, patch the image registry to use local storage:

> **NOTE**: Adding persistent storage options (NFS, iSCSI, etc) to soloshift is a work in progress 

`(util)# oc patch configs.imageregistry.operator.openshift.io cluster \`

`--type merge \`

`--patch '{"spec":{"storage":{"emptyDir":{}}}}'`

If you receive a message like "cluster does not exist" or "cluster not found", wait a bit and rerun.

And finally, watch for the "Install complete!" message, which will be followed by auth creds to log into the console...

`(util)# openshift-install --dir=/root/ocp4upi wait-for install-complete --log-level debug`

Watch for the "Install complete!" message, which will be followed by auth creds to log into the console.

You can also view the status of the bootstrap process as nodes come and go by checking out the haproxy status page at http://192.168.8.8:9000

Once the installation is complete, edit your hypervisor's /etc/hosts file to include some of the endpoints utilized by OpenShift.  For example if using all defaults, your /etc/hosts entries would look like this:

	192.168.8.8 console-openshift-console.apps.ocp42.local.dc
	192.168.8.8 oauth-openshift.apps.ocp42.local.dc
	192.168.8.8 prometheus-k8s-openshift-monitoring.apps.ocp42.local.dc
	192.168.8.8 grafana-openshift-monitoring.apps.ocp42.local.dc

In addition, you'll need to add the FQDN for any additional routes created for applications while you use OpenShift.  Utilization of wildcard DNS entries is in the works. See below for additional instructions for utilizing xip.io to enable route name resolution without additional hosts file entries.


When you're ready to tear everything down, execute:

`(hypervisor)# cd soloshift`

`(hypervisor)# ansible-playbook playbooks/99-ocp-wipe.yaml`


Utilizing xip.io for application base subdomain name resolution
------------

[Route resolution via xip.io](https://github.com/heatmiser/soloshift/blob/master/route_resolution.md)
