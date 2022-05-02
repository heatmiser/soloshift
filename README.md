# soloshift
Ansible automation for deploying a local "all in one" OpenShift 4 cluster

# Overview

While single system installation of OpenShift 4 is also made possible via [Red Hat CodeReady Containers](https://developers.redhat.com/blog/2019/09/05/red-hat-openshift-4-on-your-laptop-introducing-red-hat-codeready-containers/), some users desire a more complete single system installation of OpenShift 4 that more closely resembles a full, multi-system OpenShift 4 cluster deployment, whether that be for laptop installation or single server lab deployments.

# Cluster components

By default, soloshift deploys a 3-0-2 OpenShift 4.x cluster stack comprised of three control nodes and two worker nodes, with the zero representing optional infrastructure nodes.  In addition, a single utility node is deployed that provides DHCP, tftp, DNS, matchbox, haproxy, and NFS storage services, as well as serving as a location for the OpenShift 4 User Provided Infrastructure installation directory. If base system resources can support it, the cluster can be expanded to utilize multiple infrastructure nodes, as well as additional worker nodes.  Analyzing the available system resources to see if supporting additional nodes is possible is left as an exercise to the end user.

Requirements
------------

A Linux KVM hypervisor OS, preferably RHEL/CentOS/Fedora with a minimum of 32GB RAM, however, more is recommended. Note that work is in progress to see if installations on Macs can be supported.

Installation
------------

The system where commands are to be executed is listed in parentheses next to the shell prompt. All commands are to be executed as a non-root user with sudo capabilities, unless otherwise noted in the instructions preceding a step.

For CentOS hypervisors, skip to step 2.

For Fedora hypervisors, skip to step 3.

1) If you're using RHEL for the base hypervisor OS, then you'll need to register the system and configure the system subscription first.

As root, or via sudo:

- Red Hat username and password

`(hypervisor)# subscription-manager register --username="your_user_name" --password="your_user_password"`

> **NOTE**: Leave out the --password switch if you want to enter your password interactively and not record password in shell history.

> **NOTE**: Utilizing username/password will require a subsequent step to attach a subscription, see Red Hat documentation for requisite procedure if needed. For example:

`(hypervisor)# subscription-manager attach --pool=<pool_id_string>`

- ...or Red Hat organization ID and activation key:

`(hypervisor)# subscription-manager register --activationkey="your_key_name" --org="your_org_id#"`

...then:

| RHEL 7.x hypervisors |
|:-:|

`(hypervisor)# subscription-manager repos --disable="*"`

`(hypervisor)# subscription-manager repos --enable="rhel-7-server-rpms"`

`(hypervisor)# subscription-manager repos --enable="rhel-7-server-extras-rpms"`

`(hypervisor)# subscription-manager repos --enable="rhel-7-server-ansible-2.8-rpms"`

| RHEL 8.x hypervisors |
|:-:|

`(hypervisor)# subscription-manager repos --enable ansible-2.9-for-rhel-8-x86_64-rpms`

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

SoloShift employs a utility VM to run the OpenShift 4 User Provided Infrastructure installation, as well as to provide base infrastructure services, such as DHCP, tftp, DNS, matchbox, haproxy, and NFS storage.

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

* `ocp_vms_base_image`: rhel-server-7.8-x86_64-kvm.qcow2 - enter name of RHEL KVM Guest image downloaded from https://access.redhat.com/downloads --also, Fedora or CentOS cloud KVM qcow2 images can be used as well
* `ocp_vms_openshift_release`: 4.9 - OpenShift version to deploy; 4.1, 4.2, ..., 4.9 or pre-release (4.10 is in dev)
* `ocp_vms_openshift_subdomain`: ocp4 - name for top level DNS sub-domain
* `ocp_vms_openshift_rootdomain`: domain.com - base DNS second-level domain
* `ocp_vms_libvirt_images_location`: Using a vm image storage location different than the default?  Define it here.
* `ocp_vms_net_cidr`: 192.168.8.0/24 - internal subnet for cluster to use
* `ocp_vms_control_count`: 3
* `ocp_vms_infra_count`: 0
* `ocp_vms_worker_count`: 2
* `ocp_vms_storage_type`: nfs - default external storage type, set to false to turn it off
* `ocp_vms_openshift_pullsecret_file`: pull-secret.txt - download from https://cloud.redhat.com/openshift/install/metal/user-provisioned

By default, SoloShift deploys VMs utilizing sparse backing files.  When creating VMs, if the size of the requested backing file is greater than the total amount of space available in the volume containing the directory defined by `ocp_vms_libvirt_images_location`, the deploy automation will fail. If you would like to implement storage overcommit in order to bypass this limitation, add the following line to `inventory/group_vars/all/my_vars.yaml`:

* `ocp_vms_storage_overcommit: True`

> **NOTE**: A basic installation utilizing soloshift (with default values) to deploy an OpenShift 4 cluster comprised of 1 control, 1 infrastructure, and 1 worker, with one utility node will require just under 40GB of space to complete the installation. Additional space will be required over time as the cluster is used.

> **NOTE**: Storage overcommit can potentially lead to completely filling the total amount of space available in the volume containing the directory defined by `ocp_vms_libvirt_images_location`. Be sure to keep this in mind and monitor overall volume capacity.

Deploy All-in-One OCP4
------------

Place `pull-secret.txt` in the root of the soloshift directory.

Download your VM image of choice (RHEL8 KVM qcow2 guest image, for example) and place it in the directory defined by `ocp_vms_libvirt_images_location`. Then, update `ocp_vms_base_image` with the name of the image. If you have configured a non-standard VM images directory location, place the VM image there and make sure to update `ocp_vms_libvirt_images_location` to reflect that location.

If you'd like to adjust the number of vcpus, memory, ram, or disk sizes of the various VM nodes, edit
`roles/ocp4-solo-vmprovision/defaults/main.yml` before proceeding. The default values are as low as you should go for successful installations. A minimum base hypervisor RAM of 32GB is required, however, there is potential for VMs to be stopped if the OOM killer is enabled on the hypervisor. Laptop installation was one of the original goals for soloshift, however, the requirement for 3 control nodes has raised the miminum recommended RAM beyond the capabilities of most laptops. 48GB of RAM is more likely to ensure sucess, with 64GB RAM being a more realistic minimum RAM specification.

Now, continue to install required Ansible roles and execute OCP deploy playbooks:

`(hypervisor)# ansible-galaxy install -p ./roles -r requirements.yaml`

`(hypervisor)# ansible-playbook playbooks/00-ocp-hyper.yaml`

`(hypervisor)# ansible-playbook playbooks/01-ocp-vms.yaml`

`(hypervisor)# ansible-playbook playbooks/02-ocp-util-node.yaml`

`(hypervisor)# ansible-playbook playbooks/03-ocp-init.yaml`

> **NOTE**: While all playbooks provide output to the console to convey overall playbook progress, the 03-ocp-init.yaml playbook has additional installation log monitoring tasks that will relay the status of the deployment.

You can also view the status of the bootstrap and install process as nodes come and go by checking out the haproxy status page at http://192.168.8.8:9000

Eventually, you should see a debug message in the shell where the 03-ocp-init.yaml playbook is running: "Install complete!"

At this point, OpenShift storage needs to be finalized. Currently two options are available: external NFS storage (default) or simple ephemeral storage.

| External NFS Storage |
|:-:|
If you opted for utilizing the default option, external NFS storage on the util node, then execute the following playbook:

`(hypervisor)# ansible-playbook playbooks/04-ocp-nfs-storage.yaml`

| Simple Ephemeral Storage |
|:-:|
Or if simple, ephemeral storage is desired, prior to deploying the cluster, set the variable `ocp_vms_storage_type` to `false` and then proceed with the following steps once the 03-ocp-init.yaml playbook has finalized with the "Install complete!" message.

Either access the util vm console via virt-viewer or use another shell and ssh into the util vm as root. `ocp_vms_password` is the root password, set in the defaults for the `ocp4-solo-vmprovision` role. If you left `ocp_vms_net_cidr` at the default internal subnet to use, then the util node will be at 192.168.8.8.  There will be an SSH key pair in your user's .ssh directory prefixed with whatever was set for `ocp_vms_openshift_subdomain`.  You can use that private key to ssh in to the util node as root. For example, from a shell on the hypervisor (`ocp_vms_openshift_subdomain` set to `ocp4` and default `ocp_vms_net_cidr`):

`(hypervisor)# ssh -i ~/.ssh/ocp4_id_ecdsa root@192.168.8.8`

At this point, patch the image registry to use local storage:

> **NOTE**: Adding persistent storage options (NFS, iSCSI, etc) to soloshift is a work in progress 

	(util)# oc patch configs.imageregistry.operator.openshift.io cluster \
		--type merge \
		--patch '{"spec":{"storage":{"emptyDir":{}}}}'
	(util)# oc patch configs.imageregistry.operator.openshift.io cluster \
	    --type merge \
        --patch '{"spec":{"managementState":"Managed"}}'

If you receive a message like "cluster does not exist" or "cluster not found", wait a bit and rerun.

Final Steps
------------

Once either storage setup option has been completed, the installation is complete. Edit your hypervisor's /etc/hosts file to include some of the endpoints utilized by OpenShift.  For example if using all defaults, your /etc/hosts entries would look like this:

	192.168.8.8 console-openshift-console.apps.ocp46.local.dc
	192.168.8.8 oauth-openshift.apps.ocp46.local.dc
	192.168.8.8 prometheus-k8s-openshift-monitoring.apps.ocp46.local.dc
	192.168.8.8 grafana-openshift-monitoring.apps.ocp46.local.dc

In addition, you'll need to add the FQDN for any additional routes created for applications while you use OpenShift.  Utilization of wildcard DNS entries is in the works. See below for additional instructions for utilizing [xip.io](http://xip.io/) to enable route name resolution without additional hosts file entries.

If you plan on shutting down your OpenShift 4 cluster from time to time (laptop, cloud deployment, educational/lab), we'll need to create a DaemonSet that pulls down the same service account token bootstrap credential used on all the non-control nodes in the cluster and then delete a couple of key, related secrets. This will trigger the Cluster Operators to re-create the CSR signer secrets used to sign the kubelet client certificate CSRs when the cluster starts back up. [Follow this link](https://blog.openshift.com/enabling-openshift-4-clusters-to-stop-and-resume-cluster-vms/) for a detailed background explanation as to why we need to do this.

Execute the following steps, again, while logged in to the util node as root:

`(util)# oc apply -f $HOME/kubelet-bootstrap-cred-manager-ds.yaml`

`(util)# oc delete secrets/csr-signer-signer secrets/csr-signer -n openshift-kube-controller-manager-operator`

This will trigger the Cluster Operators to re-create the CSR signer secrets. You can watch progress as various operators switch from Progressing=False to Progressing=True and back to Progressing=False:

`(util)# watch oc get clusteroperators`

Once all Cluster Operators show Available=True, Progressing=False and Degraded=False the cluster can be safely shutdown.

If you did not re-create the CSR signer secrets used to sign the kubelet client certificate CSRs and the cluster missed the initial 24 hour certificate rotation, some nodes in the cluster may be in the NotReady state. Follow the instructions at the end of [this link](https://blog.openshift.com/enabling-openshift-4-clusters-to-stop-and-resume-cluster-vms/) to rectify.  An Ansible playbook `ocp-approve-csr.yaml` has been included in the root user's home dir that can be run as part of the rectification process.

Enjoy your OpenShift 4 cluster environment!  When you're ready to tear everything down, execute:

`(hypervisor)# cd soloshift`

`(hypervisor)# ansible-playbook playbooks/99-ocp-wipe.yaml`


Utilizing xip.io for application base subdomain name resolution
------------

[Route resolution via xip.io](https://github.com/heatmiser/soloshift/blob/main/route_resolution.md)
