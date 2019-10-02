# soloshift
Ansible automation for deploying a local "all in one" OpenShift 4 cluster

SoloShift
=========

OpenShift 4 installed on a single system

Requirements
------------

A Linux KVM hypervisor OS, preferably RHEL/CentOS/Fedora.

`# git clone this repo`

Edit `inventory/group_vars/all/default_vars.yaml`

Choose either pair of:

- organization ID and activation key
  
or

- username and password 

...and leave the other pair undefined.

* `redhat_subscription_org_id`: Subscription organization ID. If using an activation key, this is required. If comprised of all numbers, surround in double-quotes.
* `redhat_subscription_activationkey`: Activation key to use for host registration.


* `redhat_subscription_username`: If not using an activation key, specify Red Hat username. 
* `redhat_subscription_password`: If not using an activation key, specify Red Hat password.

* `ocp_vms_base_image`: rhel-server-7.7-x86_64-kvm.qcow2
* `ocp_vms_openshift_release`: ocp41
* `ocp_vms_openshift_subdomain`: domain.com
* `ocp_vms_libvirt_images_location`: /u01/libvirt/images
* `ocp_vms_net_cidr`: 192.168.8.0/24
* `ocp_vms_master_count`: 1
* `ocp_vms_infra_count`: 1
* `ocp_vms_worker_count`: 1
* `ocp_vms_openshift_pullsecret_file`: pull-secret.txt


`# ansible-galaxy install --role-path ./roles -r requirements.yaml`

`# ansible-playbook playbooks/00-ocp-hyper.yaml`

`# ansible-playbook playbooks/01-ocp-vms.yaml`

`# ansible-playbook playbooks/02-ocp-util.yaml`

`# ansible-playbook playbooks/03-ocp-init.yaml`

Either access the util vm console via virt-viewer or ssh into the util vm as root

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