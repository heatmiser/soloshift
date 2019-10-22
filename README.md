# soloshift
Ansible automation for deploying a local "all in one" OpenShift 4 cluster

Requirements
------------

A Linux KVM hypervisor OS, preferably RHEL/CentOS/Fedora with a minimum of 16GB RAM. If you're using RHEL for the base hypervisor OS, then you'll need to register and configure first.  As root, or via sudo:

`# subscription-manager register --username="your_user_name" --password="your_user_password"`
Note: Leave out the --password switch if you want to enter your password interactively and not record password in shell history.

...or:

`# subscription-manager register --activationkey="your_key_name" --org="your_org_id#"`

...then:

`# subscription-manager repos --disable="*"`

`# subscription-manager repos --enable="rhel-7-server-rpms"`

`# subscription-manager repos --enable="rhel-7-server-extras-rpms"`

`# subscription-manager repos --enable="rhel-7-server-ansible-2.8-rpms"`

`# yum install git ansible`

`# git clone https://github.com/heatmiser/soloshift.git`

`# cd soloshift`

Copy `inventory/group_vars/all/default_vars.yaml`
to `inventory/group_vars/all/my_vars.yaml`


If utilizing RHEL for the base hypervisor system, edit `inventory/group_vars/all/my_vars.yaml` and choose either pair of:

- organization ID and activation key
  
or

- username and password 

...and leave the other pair undefined.

* `redhat_subscription_org_id`: Subscription organization ID. If using an activation key, this is required. If comprised of all numbers, surround in double-quotes.
* `redhat_subscription_activationkey`: Activation key to use for host registration.


* `redhat_subscription_username`: If not using an activation key, specify Red Hat username. 
* `redhat_subscription_password`: If not using an activation key, specify Red Hat password.

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

Deploy All-in-One OCP4
------------

Place pull-secret.txt in the root of the soloshift directory.

Download your VM image of choice to /var/lib/libvirt/images, for example, the RHEL7 KVM qcow2 guest image. Then, update `ocp_vms_base_image` with the name of the image.  If you have configured a non-standard VM images directory location, place the VM image there and make sure to update `ocp_vms_libvirt_images_location` to reflect that location.

`# ansible-galaxy install -p ./roles -r requirements.yaml`

`# ansible-playbook playbooks/00-ocp-hyper.yaml`

`# ansible-playbook playbooks/01-ocp-vms.yaml`

`# ansible-playbook playbooks/02-ocp-util.yaml`

`# ansible-playbook playbooks/03-ocp-init.yaml`

Either access the util vm console via virt-viewer or ssh into the util vm as root. `ocp_vms_password` is the root password, set in the defaults for the `ocp4-solo-vmprovision` role. If you left `ocp_vms_net_cidr` at the default internal subnet to use, then the util node will be at 192.168.8.8.  There will be an SSH key pair in your user's .ssh directory prefixed with whatever was set for `ocp_vms_openshift_subdomain`.  You can use that private key to ssh in to the util node as root.

After logging in to the util node as root, execute:

`# openshift-install --dir=/root/ocp4upi wait-for bootstrap-complete --log-level debug`

Eventually, you'll see a log message saying that it's ok to shutdown the bootstrap machine. Back on the hypervisor system, shutdown the bootstrap node, either via the Virtual Machine Manager or `virsh` command line tool.

Next, patch the image registry to use local storage:

`oc patch configs.imageregistry.operator.openshift.io cluster \`

`--type merge \`

`--patch '{"spec":{"storage":{"emptyDir":{}}}}'`

If you receive a message like "cluster does not exist" wait a bit and rerun.

And finally, watch for the "Install complete!" message, which will be followed by auth creds to log into the console...

`openshift-install --dir=/root/ocp4upi wait-for install-complete --log-level debug`

You can also view the status of the bootstrap process as nodes come and go by checking out the haproxy status page at http://192.168.8.8:9000

Once the installation is complete, edit your hypervisor's /etc/hosts file to include some of the endpoints utilized by OpenShift.  For example if using all defaults, your /etc/hosts entries would look like this:

192.168.8.8 console-openshift-console.apps.ocp42.local.dc
192.168.8.8 oauth-openshift.apps.ocp42.local.dc
192.168.8.8 prometheus-k8s-openshift-monitoring.apps.ocp42.local.dc
192.168.8.8 grafana-openshift-monitoring.apps.ocp42.local.dc

You'll need to add the FQDN for any additional routes created for applications while you use OpenShift.  Utilization of wildcard DNS entries is in the works...


When you're ready to tear everything down, execute:

`ansible-playbook playbooks/99-ocp-wipe.yaml`