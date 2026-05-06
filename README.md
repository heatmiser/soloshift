# soloshift
Ansible automation for deploying a local "all in one" OpenShift 4 cluster

# Overview

While single system installation of OpenShift 4 is also made possible via [Red Hat OpenShift Local](https://developers.redhat.com/products/openshift-local/overview), some users desire a more complete single system installation of OpenShift 4 that more closely resembles a full, multi-system OpenShift 4 cluster deployment, whether that be for server lab deployments or development environments.

# Cluster components

By default, soloshift deploys a 3-0-2 OpenShift 4.x cluster stack comprised of three control-plane nodes and two worker nodes, with the zero representing optional infrastructure nodes. In addition, a single utility node is deployed that provides DHCP, tftp, DNS, matchbox, haproxy, and NFS storage services, as well as serving as a location for the OpenShift 4 User Provided Infrastructure installation directory. If base system resources can support it, the cluster can be expanded to utilize multiple infrastructure nodes, as well as additional worker nodes. Analyzing the available system resources to see if supporting additional nodes is possible is left as an exercise to the end user.

# Requirements

A Linux KVM hypervisor running **RHEL 8/9 or Fedora** with a minimum of 48GB RAM; 64GB or more is strongly recommended for stable operation of all three control-plane nodes plus workers. The utility node runs RHEL 9 by default.

> **NOTE**: CentOS 7 and RHEL 7 are no longer supported hypervisor or utility node operating systems.

# Installation

The system where commands are to be executed is listed in parentheses next to the shell prompt. All commands are to be executed as a non-root user with sudo capabilities, unless otherwise noted.

## 1. Register and configure the hypervisor (RHEL only)

As root, or via sudo, register using an organization ID and activation key:

```
(hypervisor)# subscription-manager register --activationkey="your_key_name" --org="your_org_id"
```

Or using username and password:

```
(hypervisor)# subscription-manager register --username="your_user_name" --password="your_user_password"
(hypervisor)# subscription-manager attach --pool=<pool_id_string>
```

Enable the required repositories:

**RHEL 8:**
```
(hypervisor)# subscription-manager repos --disable="*"
(hypervisor)# subscription-manager repos --enable="rhel-8-for-x86_64-baseos-rpms"
(hypervisor)# subscription-manager repos --enable="rhel-8-for-x86_64-appstream-rpms"
```

**RHEL 9:**
```
(hypervisor)# subscription-manager repos --disable="*"
(hypervisor)# subscription-manager repos --enable="rhel-9-for-x86_64-baseos-rpms"
(hypervisor)# subscription-manager repos --enable="rhel-9-for-x86_64-appstream-rpms"
```

## 2. Install Ansible and git

**RHEL 8/9:**
```
(hypervisor)# dnf -y install git ansible-core
```

**Fedora:**
```
(hypervisor)# dnf -y install git ansible-core
```

## 3. Clone the repository

```
(hypervisor)# git clone https://github.com/heatmiser/soloshift.git
(hypervisor)# cd soloshift
```

## 4. Configure your deployment variables

Copy the default variables file to create your local overrides file:

```
(hypervisor)# cp inventory/group_vars/all/default_vars.yaml inventory/group_vars/all/my_vars.yaml
```

Edit `inventory/group_vars/all/my_vars.yaml` and set the following variables:

### Red Hat subscription credentials (for the utility VM)

Choose either organization ID + activation key **or** username + password — leave the other pair undefined.

- `redhat_subscription_org_id`: Subscription organization ID (required when using an activation key). If the value is all digits, surround it in double quotes.
- `redhat_subscription_activationkey`: Activation key to use for host registration.
- `redhat_subscription_username`: Red Hat username (if not using an activation key).
- `redhat_subscription_password`: Red Hat password (if not using an activation key).
- `redhat_subscription_pool_regex`: If using username/password, supply a regex to match the desired subscription pool. For example: `"^Red Hat Enterprise Linux$"`

### Core deployment variables

- `ocp_vms_base_image`: Name of the RHEL 9 KVM guest image downloaded from https://access.redhat.com/downloads. Example: `rhel-9.4-x86_64-kvm.qcow2`. Place the image in the directory defined by `ocp_vms_libvirt_images_location`.
- `ocp_vms_password`: Root password for the utility VM. **Required** — no default is set.
- `ocp_vms_openshift_release`: OpenShift version to deploy. Example: `"4.16"`.
- `ocp_vms_openshift_subdomain`: Top-level DNS sub-domain name. Example: `ocp416`.
- `ocp_vms_openshift_rootdomain`: Base DNS second-level domain. Example: `local.dc`.
- `ocp_vms_libvirt_images_location`: VM image storage directory. Default: `/var/lib/libvirt/images/`.
- `ocp_vms_net_cidr`: Internal subnet for the cluster. Default: `192.168.8.0/24`.
- `ocp_vms_master_count`: Number of control plane nodes. Default: `3`.
- `ocp_vms_infra_count`: Number of infrastructure nodes. Default: `0`.
- `ocp_vms_worker_count`: Number of worker nodes. Default: `2`.
- `ocp_vms_storage_type`: External storage type. Default: `nfs`. Set to `false` for ephemeral storage.
- `ocp_vms_openshift_pullsecret_file`: Pull secret filename. Default: `pull-secret.txt`. Download from https://console.redhat.com/openshift/install/metal/user-provisioned.

### Storage space considerations

By default, soloshift deploys VMs utilizing sparse backing files. If the size of the requested backing file exceeds the total available space in the volume defined by `ocp_vms_libvirt_images_location`, the deploy will fail. To enable storage overcommit:

```yaml
ocp_vms_storage_overcommit: true
```

> **NOTE**: A minimal soloshift installation (1 control-plane, 1 infra, 1 worker, 1 utility node) requires approximately 40GB of disk space to complete. Additional space is consumed over time as the cluster is used.

> **NOTE**: Storage overcommit can lead to completely filling the volume. Monitor capacity accordingly.

If you'd like to adjust vCPU counts, memory, or disk sizes, edit `roles/ocp4_solo_vmprovision/defaults/main.yml` before proceeding. The default values represent the minimum viable configuration. A minimum of 48GB hypervisor RAM is recommended; 64GB significantly improves stability.

# Deploy All-in-One OCP4

Place `pull-secret.txt` in the root of the soloshift directory.

Download the RHEL 9 KVM guest image from https://access.redhat.com/downloads and place it in the directory defined by `ocp_vms_libvirt_images_location`. Update `ocp_vms_base_image` in `my_vars.yaml` with the exact filename.

Install required Ansible roles and collections, then execute the deployment playbooks in order:

```
(hypervisor)# ansible-galaxy role install -p ./roles -r requirements.yaml
(hypervisor)# ansible-galaxy collection install -r requirements.yaml

(hypervisor)# ansible-playbook playbooks/00-ocp-hyper.yaml
(hypervisor)# ansible-playbook playbooks/01-ocp-vms.yaml
(hypervisor)# ansible-playbook playbooks/02-ocp-util-node.yaml
(hypervisor)# ansible-playbook playbooks/03-ocp-init.yaml
```

> **NOTE**: The `03-ocp-init.yaml` playbook includes log monitoring tasks that relay the status of the bootstrap and installation process. You can also monitor progress via the haproxy status page at `http://<util_node_ip>:9000` (default: `http://192.168.8.8:9000`).

When the `03-ocp-init.yaml` playbook finishes with "Install complete!", proceed to storage configuration.

# Storage Configuration

## External NFS Storage (default)

```
(hypervisor)# ansible-playbook playbooks/04-ocp-nfs-storage.yaml
```

## Simple Ephemeral Storage

If ephemeral storage is desired, set `ocp_vms_storage_type: false` in `my_vars.yaml` before deploying. Once `03-ocp-init.yaml` has finished, SSH into the utility VM and patch the image registry:

```
(hypervisor)# ssh -i ~/.ssh/<ocp_vms_openshift_subdomain>_id_ecdsa root@192.168.8.8
```

```
(util)# oc patch configs.imageregistry.operator.openshift.io cluster \
    --type merge \
    --patch '{"spec":{"storage":{"emptyDir":{}}}}'
(util)# oc patch configs.imageregistry.operator.openshift.io cluster \
    --type merge \
    --patch '{"spec":{"managementState":"Managed"}}'
```

If you receive "cluster does not exist", wait a moment and retry.

# Final Steps

Once storage is configured, add cluster endpoints to your hypervisor's `/etc/hosts`. Using default values with `ocp_vms_openshift_subdomain: ocp416` and `ocp_vms_openshift_rootdomain: local.dc`:

```
192.168.8.8 console-openshift-console.apps.ocp416.local.dc
192.168.8.8 oauth-openshift.apps.ocp416.local.dc
192.168.8.8 prometheus-k8s-openshift-monitoring.apps.ocp416.local.dc
192.168.8.8 grafana-openshift-monitoring.apps.ocp416.local.dc
```

Add additional entries for any application routes created during cluster use.

# Cluster Shutdown and Restart Support

If you plan on shutting down your cluster periodically, you must prepare it to handle certificate rotation on restart. This requires deploying a DaemonSet that refreshes the kubelet bootstrap credential and rotating the CSR signer secrets. See [this post](https://blog.openshift.com/enabling-openshift-4-clusters-to-stop-and-resume-cluster-vms/) for background.

While logged into the utility VM as root:

```
(util)# oc apply -f $HOME/kubelet-bootstrap-cred-manager-ds.yaml
(util)# oc delete secrets/csr-signer-signer secrets/csr-signer -n openshift-kube-controller-manager-operator
```

Watch Cluster Operators recover:

```
(util)# watch oc get clusteroperators
```

Once all operators show `Available=True, Progressing=False, Degraded=False`, the cluster can be safely shut down.

If the cluster missed the initial 24-hour certificate rotation after a previous unclean shutdown, some nodes may be in `NotReady` state. Follow the recovery steps at the link above. An Ansible playbook `ocp-approve-csr.yaml` is included in the root user's home directory to assist with the CSR approval process.

# Tear Down

```
(hypervisor)# ansible-playbook playbooks/99-ocp-wipe.yaml
```

# Route Resolution

[Route resolution via xip.io](https://github.com/heatmiser/soloshift/blob/master/route_resolution.md)
