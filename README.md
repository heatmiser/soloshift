# soloshift
Ansible automation for deploying a local "all in one" OpenShift 4 cluster

Requirements
------------

A Linux KVM hypervisor OS, preferably RHEL/CentOS/Fedora.

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
* `ocp_vms_openshift_release`: ocp41 - name for top level DNS sub-domain
* `ocp_vms_openshift_subdomain`: domain.com - base DNS subdomain
* `ocp_vms_libvirt_images_location`: Using a vm image storage location different than the default?  Define it here.
* `ocp_vms_net_cidr`: 192.168.8.0/24 - internal subnet for cluster to use
* `ocp_vms_master_count`: 1
* `ocp_vms_infra_count`: 1
* `ocp_vms_worker_count`: 1
* `ocp_vms_openshift_pullsecret_file`: pull-secret.txt - download from https://cloud.redhat.com/openshift/install/metal/user-provisioned

Deploy All-in-One OCP4
------------

`# ansible-galaxy install -p ./roles -r requirements.yaml`

`# ansible-playbook playbooks/00-ocp-hyper.yaml`

`# ansible-playbook playbooks/01-ocp-vms.yaml`

`# ansible-playbook playbooks/02-ocp-util.yaml`

`# ansible-playbook playbooks/03-ocp-init.yaml`

Either access the util vm console via virt-viewer or ssh into the util vm as root and execute:

`# openshift-install --dir=/root/ocp4upi wait-for bootstrap-complete --log-level debug`

Eventually, you'll see a log message saying that it's ok to shutdown the bootstrap machine, then do that.

Next, patch the image registry to use local storage:

`oc patch configs.imageregistry.operator.openshift.io cluster \`

`--type merge \`

`--patch '{"spec":{"storage":{"emptyDir":{}}}}'`

And finally, watch for the "Install complete!" message, which will be followed by auth creds to log into the console...

`openshift-install --dir=/root/ocp4upi wait-for install-complete --log-level debug`

When ready to tear everything down, execute:

`ansible-playbook playbooks/99-ocp-wipe.yaml`