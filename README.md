# OpenShift 4 Bare Metal Install - User Provisioned Infrastructure (UPI)

- [OpenShift 4 Bare Metal Install - User Provisioned Infrastructure (UPI)](#openshift-4-bare-metal-install---user-provisioned-infrastructure-upi)
  - [Architecture Diagram](#architecture-diagram)
  - [Download Software](#download-software)
  - [Prepare the 'Bare Metal' environment](#prepare-the-bare-metal-environment)
  - [Configure Environmental Services](#configure-environmental-services)
  - [Generate and host install files](#generate-and-host-install-files)
  - [Deploy OpenShift](#deploy-openshift)
  - [Monitor the Bootstrap Process](#monitor-the-bootstrap-process)
  - [Remove the Bootstrap Node](#remove-the-bootstrap-node)
  - [Wait for installation to complete](#wait-for-installation-to-complete)
  - [Join Worker Nodes](#join-worker-nodes)
  - [Configure storage for the Image Registry](#configure-storage-for-the-image-registry)
  - [Create the first Admin user](#create-the-first-admin-user)
  - [Access the OpenShift Console](#access-the-openshift-console)
  - [Troubleshooting](#troubleshooting)
  - [Testing APP](#testing-app)


## Architecture Diagram

![Architecture Diagram](Architecture.png)


## Download Software
1. Download [CentOS 8 x86_64 image](https://www.centos.org/centos-linux/)
1. Login to [RedHat OpenShift Cluster Manager](https://cloud.redhat.com/openshift)
1. Select 'Create Cluster' from the 'Clusters' navigation menu
1. Select 'RedHat OpenShift Container Platform'
1. Select 'Run on Bare Metal'
1. Download the following files:

   - Openshift Installer for Linux
   - Pull secret
   - Command Line Interface 
   - Red Hat Enterprise Linux CoreOS (RHCOS)
     - rhcos-X.X.X-x86_64-installer.x86_64.iso

## Prepare the 'Bare Metal' environmen
> VMware ESXi used in this guide

1. Copy the CentOS 8 and rhcos-installer.x86_64 iso to an ESXi datastore
1. Create a new Port Group called 'OCP' under Networking
1. Create 3 Control Plane virtual machines with minimum settings:
   - Name: master01.ocp-dev.datacomm.co.id
   - 4vcpu
   - 16GB RAM
   - 120GB HDD
   - NIC connected to the OCP network (DHCP)
   - Load the rhcos-X.X.X-x86_64-installer.x86_64.iso image into the CD/DVD drive
1. Create 2 Worker virtual machines (or more if you want) with minimum settings:
   - Name: worker01.ocp-dev.datacomm.co.id
   - 2vcpu
   - 8GB RAM
   - 120GB HDD
   - NIC connected to the OCP network (DHCP)
   - Load the rhcos-X.X.X-x86_64-installer.x86_64.iso image into the CD/DVD drive
1. Create a Bootstrap virtual machine (this vm will be deleted once installation completes) with minimum settings:
   - Name: bootstrap.ocp-dev.datacomm.co.id
   - 4vcpu
   - 8GB RAM
   - 120GB HDD
   - NIC connected to the OCP network (DHCP)
   - Load the rhcos-X.X.X-x86_64-installer.x86_64.iso image into the CD/DVD drive
1. Create a helper virtual machine with minimum settings:
   - Name: helper.ocp-dev.datacomm.co.id
   - 4vcpu
   - 8GB RAM
   - 120GB HDD1
   - 120GB HDD2
   - NIC1 connected to the OCP network (NAT Point to Point IP public) 10.19.15.3
   - Load the CentOS_8.iso image into the CD/DVD drive
1. Boot all virtual machines so they each are assigned a MAC address
1. Shut down all virtual machines except for 'helper.ocp-dev.datacomm.co.id'
1. Use the VMware ESXi dashboard to record the MAC address of each vm, these will be used later to set static IPs


## Configure Environmental Services

1. Install CentOS8 on the helper.ocp-dev.datacomm.co.id host
   - Remove the home dir partition and assign all free storage to '/'
   - Set Static IP addreess 10.19.15.3 to NIC 1
   - DNS1 Server 127.0.0.1
   - DNS2 Server 8.8.8.8
1. Boot the helper.ocp-dev.datacomm.co.id VM

1. Move the files downloaded from the RedHat Cluster Manager site to the helper.ocp-dev.datacomm.co.id node

   ```bash
   scp ~/Downloads/openshift-install-linux.tar.gz ~/Downloads/openshift-client-linux.tar.gz  root@{helper.ocp-dev.datacomm.co.id_IP_address}:/root/
   ```

1. SSH to the ocp-svc vm

   ```bash
   ssh root@{helper_IP_address}
   ```

1. Extract Client tools and copy them to `/usr/local/bin`

   ```bash
   tar xvf openshift-client-linux.tar.gz
   mv oc kubectl /usr/local/bin
   ```

1. Confirm Client Tools are working

   ```bash
   kubectl version
   oc version
   ```

1. Extract the OpenShift Installer

   ```bash
   tar xvf openshift-install-linux.tar.gz
   ```

1. Update CentOS so we get the latest packages for each of the services we are about to install

   ```bash
   dnf update
   ```
   
1. Install Git

   ```bash
   dnf install git -y
   ```

1. Download [config files](https://github.com/teghitsugaya/ocp4.6-baremetal-vmware) for each of the services

   ```bash
   git clone https://github.com/ryanhay/ocp4-metal-install
   ```
   
1. OPTIONAL: Create a file '~/.vimrc' and paste the following (this helps with editing in vim, particularly yaml files):

   ```bash
   cat <<EOT >> ~/.vimrc
   syntax on
   set nu et ai sts=0 ts=2 sw=2 list hls
   EOT
   ```

   Update the preferred editor

   ```bash
   export OC_EDITOR="vim"
   export KUBE_EDITOR="vim"
   ```

1. Install and configure BIND DNS

   Install

   ```bash
   dnf install bind bind-utils -y
   ```

   Apply configuration

   ```bash
   cp ~/ocp4.6-baremetal-vmware/named.conf /etc/named.conf
   cp -R ~/ocp4.6-baremetal-vmware/db* /var/named/.
   ```

   Configure the firewall for DNS

   ```bash
   firewall-cmd --add-port=53/udp --zone=internal --permanent
   firewall-cmd --reload
   ```

   Enable and start the service

   ```bash
   systemctl enable named
   systemctl start named
   systemctl status named
   ```

   Confirm dig now sees the correct DNS results by using the DNS Server running locally

   ```bash
   dig -x 10.19.15.200
   nslookup api.cluster-jkt01.ocp-dev.datacomm.co.id
   ```
1. Install & configure DHCP

   Install the DHCP Server

   ```bash
   dnf install dhcp-server -y
   ```

   Edit dhcpd.conf from the cloned git repo to have the correct mac address for each host and copy the conf file to the correct location for the DHCP service to use

   ```bash
   cp ~/ocp4.6-baremetal-vmware/dhcpd.conf /etc/dhcp/dhcpd.conf
   ```

   Configure the Firewall

   ```bash
   firewall-cmd --add-service=dhcp --zone=internal --permanent
   firewall-cmd --reload
   ```

   Enable and start the service

   ```bash
   systemctl enable dhcpd
   systemctl start dhcpd
   systemctl status dhcpd
   ```

1. Install & configure Apache Web Server

   Install Apache

   ```bash
   dnf install httpd -y
   ```

   Change default listen port to 8080 in httpd.conf

   ```bash
   sed -i 's/Listen 80/Listen 0.0.0.0:8080/' /etc/httpd/conf/httpd.conf
   ```

   Configure the firewall for Web Server traffic

   ```bash
   firewall-cmd --add-port=8080/tcp --zone=internal --permanent
   firewall-cmd --reload
   ```

   Enable and start the service

   ```bash
   systemctl enable httpd
   systemctl start httpd
   systemctl status httpd
   ```

   Making a GET request to localhost on port 8080 should now return the default Apache webpage

   ```bash
   curl localhost:8080
   ```

1. Install & configure HAProxy

   Install HAProxy

   ```bash
   dnf install haproxy -y
   ```

   Copy HAProxy config

   ```bash
   cp ~/ocp4.6-baremetal-vmware/haproxy.cfg /etc/haproxy/haproxy.cfg
   ```

   Configure the Firewall

   > Note: Opening port 9000 in the external zone allows access to HAProxy stats that are useful for monitoring and troubleshooting. The UI can be accessed at: `http://{ocp-svc_IP_address}:9000/stats`

   ```bash
   firewall-cmd --add-port=6443/tcp --zone=internal --permanent # kube-api-server on control plane nodes
   firewall-cmd --add-port=6443/tcp --zone=external --permanent # kube-api-server on control plane nodes
   firewall-cmd --add-port=22623/tcp --zone=internal --permanent # machine-config server
   firewall-cmd --add-service=http --zone=internal --permanent # web services hosted on worker nodes
   firewall-cmd --add-service=http --zone=external --permanent # web services hosted on worker nodes
   firewall-cmd --add-service=https --zone=internal --permanent # web services hosted on worker nodes
   firewall-cmd --add-service=https --zone=external --permanent # web services hosted on worker nodes
   firewall-cmd --add-port=9000/tcp --zone=external --permanent # HAProxy Stats
   firewall-cmd --reload
   ```

   Enable and start the service

   ```bash
   setsebool -P haproxy_connect_any 1 # SELinux name_bind access
   systemctl enable haproxy
   systemctl start haproxy
   systemctl status haproxy
   ```

1. Install and configure NFS for the OpenShift Registry. It is a requirement to provide storage for the Registry, emptyDir can be specified if necessary.

   Install NFS Server

   ```bash
   dnf install nfs-utils -y
   ```

   Create the Share

   Check available disk space and its location `df -h`

   ```bash
   mkdir -p /shares/registry
   chown -R nobody:nobody /shares/registry
   chmod -R 777 /shares/registry
   ```

   Export the Share

   ```bash
   echo "/shares/registry  10.19.15.0/24(rw,sync,root_squash,no_subtree_check,no_wdelay)" > /etc/exports
   exportfs -rv
   ```

   Set Firewall rules:

   ```bash
   firewall-cmd --zone=internal --add-service mountd --permanent
   firewall-cmd --zone=internal --add-service rpc-bind --permanent
   firewall-cmd --zone=internal --add-service nfs --permanent
   firewall-cmd --reload
   ```

   Enable and start the NFS related services

   ```bash
   systemctl enable nfs-server rpcbind
   systemctl start nfs-server rpcbind nfs-mountd
   ```

## Generate and host install files

1. Generate an SSH key pair keeping all default options

   ```bash
   ssh-keygen -t rsa -b 4096 -N '' -f ~/.ssh/id_rsa
   ```

1. Create an install directory

   ```bash
   mkdir ~/ocp-install
   ```

1. Copy the install-config.yaml included in the clones repository to the install directory

   ```bash
   cp ~/ocp4.6-baremetal-vmware/install-config.yaml ~/ocp-install
   ```

1. Update the install-config.yaml with your own pull-secret and ssh key.
  
   ```bash
   vim ~/ocp-install/install-config.yaml
   ```

   - Line 23 should contain the contents of your pull-secret.txt
   - Line 24 should contain the contents of your '~/.ssh/id_rsa.pub'

 

1. Generate Kubernetes manifest files

   ```bash
   ~/openshift-install create manifests --dir ~/ocp-install
   ```

   > A warning is shown about making the control plane nodes schedulable. It is up to you if you want to run workloads on the Control Plane nodes. If you dont want to you can disable this with:
   > `sed -i 's/mastersSchedulable: true/mastersSchedulable: false/' ~/ocp-install/manifests/cluster-scheduler-02-config.yml`.
   > Make any other custom changes you like to the core Kubernetes manifest files.

   Generate the Ignition config and Kubernetes auth files

   ```bash
   ~/openshift-install create ignition-configs --dir ~/ocp-install/
   ```

1. Copy all generated install files to the new web server directory

   ```bash
   cp -R ~/ocp-install/* /var/www/html/
   ```

1. Change ownership and permissions of the web server directory

   ```bash
   chcon -R -t httpd_sys_content_t /var/www/html/
   chown -R apache: /var/www/html/
   chmod 755 /var/www/html/
   ```

1. Confirm you can see all files added to the `/var/www/html/ocp4/` dir through Apache

   ```bash
   curl localhost:8080/ocp4/
   ```

## Deploy OpenShift

1. Power on the bootstrap and Master host Enter the following configuration:

   ```bash
   # Bootstrap Node
   coreos-installer install --insecure-ignition --ignition-url=http://10.19.15.3:8080/bootstrap.ign/ /dev/sda
   ```

   ```bash
   # Each of the Control Plane Nodes
   coreos-installer install --insecure-ignition --ignition-url=http://10.19.15.3:8080/master.ign/ /dev/sda
   ```

1. Power on the worker hosts 

   ```bash
   # Each of the Worker Nodes 
   coreos-installer install --insecure-ignition --ignition-url=http://10.19.15.3:8080/master.ign/ /dev/sda
   ```

## Monitor the Bootstrap Process

1. You can monitor the bootstrap process from the ocp-svc host at different log levels (debug, error, info)

   ```bash
   ~/openshift-install --dir ~/ocp-install wait-for bootstrap-complete --log-level=debug
   ```

1. Once bootstrapping is complete the ocp-boostrap node [can be removed](#remove-the-bootstrap-node)



## Remove the Bootstrap Node

1. Remove all references to the `bootstrap` host from the `/etc/haproxy/haproxy.cfg` file

   ```bash
   vim /etc/haproxy/haproxy.cfg
   #server      bootstrap.ocp-dev.datacomm.co.id 10.19.15.200:6443 check
   #server      bootstrap.ocp-dev.datacomm.co.id 10.19.15.200:22623 check
   systemctl restart haproxy
   ```

1. The bootstrap host can now be safely shutdown and deleted from the VMware ESXi Console, the host is no longer required


## Wait for installation to complete

> IMPORTANT: if you set mastersSchedulable to false the [worker nodes will need to be joined to the cluster](#join-worker-nodes) to complete the installation. This is because the OpenShift Router will need to be scheduled on the worker nodes and it is a dependency for cluster operators such as ingress, console and authentication.

1. Collect the OpenShift Console address and kubeadmin credentials from the output of the install-complete event

   ```bash
   ~/openshift-install --dir ~/ocp-install wait-for install-complete
   ```

1. Continue to join the worker nodes to the cluster in a new tab whilst waiting for the above command to complete

## Join Worker Nodes

1. Setup 'oc' and 'kubectl' clients on the ocp-svc machine

   ```bash
   export KUBECONFIG=~/ocp-install/auth/kubeconfig
   # Test auth by viewing cluster nodes
   oc get nodes
   ```

1. View and approve pending CSRs

   > Note: Once you approve the first set of CSRs additional 'kubelet-serving' CSRs will be created. These must be approved too.
   > If you do not see pending requests wait until you do.

   ```bash
   # View CSRs
   oc get csr
   # Approve all pending CSRs
   oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve
   # Wait for kubelet-serving CSRs and approve them too with the same command
   oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve
   ```

1. Watch and wait for the Worker Nodes to join the cluster and enter a 'Ready' status

   > This can take 5-10 minutes

   ```bash
   watch oc get nodes
   watch oc get clusterversion
   watch oc get clusteroperators
   ```
   
## Configure storage for the Image Registry

> A Bare Metal cluster does not by default provide storage so the Image Registry Operator bootstraps itself as 'Removed' so the installer can complete. As the installation has now completed storage can be added for the Registry and the operator updated to a 'Managed' state.

1. Create the 'image-registry-storage' PVC by updating the Image Registry operator config by updating the management state to 'Managed' and adding 'pvc' and 'claim' keys in the storage key:

   ```bash
   oc edit configs.imageregistry.operator.openshift.io
   ```

   ```yaml
   managementState: Managed
   ```

   ```yaml
   storage:
     pvc:
       claim: # leave the claim blank
   ```

1. Confirm the 'image-registry-storage' pvc has been created and is currently in a 'Pending' state

   ```bash
   oc get pvc -n openshift-image-registry
   ```

1. Create the persistent volume for the 'image-registry-storage' pvc to bind to

   ```bash
   oc create -f ~/ocp4.6-baremetal-vmware/manifest/registry-pv.yaml
   ```

1. After a short wait the 'image-registry-storage' pvc should now be bound

   ```bash
   oc get pvc -n openshift-image-registry
   ```

## Create OAuth and Admin user

1. Apply the `oauth-htpasswd.yaml` file to the cluster

   > This will create a user 'admin' with the password 'password'. To set a different username and password substitue the htpasswd key in the '~/ocp4.6-baremetal-vmware/manifest/oauth-htpasswd.yaml' file with the output of `htpasswd -n -B -b <username> <password>`

   ```bash
   oc apply -f ~/ocp4.6-baremetal-vmware/oauth-htpasswd.yaml
 
   dnf -y install httpd-tools
   htpasswd -c -B -b htpasswd admin password
   htpasswd -B -b htpasswd teguh.imanto-datacomm password
   cat htpasswd
   oc create secret generic htpasswd-secret --from-file htpasswd=~/htpasswd -n openshift-config
   oc extract secret/htpasswd-secret -n openshift-config --to - > temp
   cat temp
   ```
1. Assign the new user (admin) admin permissions

   ```bash
   oc adm policy add-cluster-role-to-user cluster-admin admin
   oc adm policy add-cluster-role-to-user cluster-admin teguh.imanto-datacomm
   ```

## Access the OpenShift Console

1. Wait for the 'console' Cluster Operator to become available

   ```bash
   oc get co
   ```

1. Append the following to your local workstations `/etc/hosts` file:

   > From your local workstation
   > If you do not want to add an entry for each new service made available on OpenShift you can configure the ocp-svc DNS server to serve externally and create a wildcard entry for \*.apps.cluster-jkt01.ocp-dev.datacomm.co.id

   ```bash
   # Open the hosts file
   sudo vi /etc/hosts

   # Append the following entries:
   156.0.100.6 helper helper.ocp-dev.datacomm.co.id api.cluster-jkt01.ocp-dev.datacomm.co.id console-openshift-console.apps.cluster-jkt01.ocp-dev.datacomm.co.id oauth-openshift.apps.cluster-jkt01.ocp-dev.datacomm.co.id downloads-openshift-console.apps.cluster-jkt01.ocp-dev.datacomm.co.id alertmanager-main-openshift-monitoring.apps.cluster-jkt01.ocp-dev.datacomm.co.id grafana-openshift-monitoring.apps.cluster-jkt01.ocp-dev.datacomm.co.id prometheus-k8s-openshift-monitoring.apps.cluster-jkt01.ocp-dev.datacomm.co.id thanos-querier-openshift-monitoring.apps.cluster-jkt01.ocp-dev.datacomm.co.id
   ```

1. Navigate to the [OpenShift Console URL](https://console-openshift-console.apps.lab.ocp.lan) and log in as the 'admin' user

   > You will get self signed certificate warnings that you can ignore
   > If you need to login as kubeadmin and need to the password again you can retrieve it with: `cat ~/ocp-install/auth/kubeadmin-password`



## Troubleshooting

1. You can collect logs from all cluster hosts by running the following command from the 'ocp-svc' host:

   ```bash
   ./openshift-install gather bootstrap --dir ocp-install --bootstrap=10.19.15.200 --master=10.19.15.201 --master=10.19.15.202 --master=10.19.15.203
   ```

1. Modify the role of the Control Plane Nodes

   If you would like to schedule workloads on the Control Plane nodes apply the 'worker' role by changing the value of 'mastersSchedulable' to true.

   If you do not want to schedule workloads on the Control Plane nodes remove the 'worker' role by changing the value of 'mastersSchedulable' to false.

   > Remember depending on where you host your workloads you will have to update HAProxy to include or exclude the control plane nodes from the ingress backends.

   ```bash
   oc edit schedulers.config.openshift.io cluster
   ```

## Reference:
   - https://www.youtube.com/watch?v=d03xg2PKOPg

## Testing APP
1. Create a project to host your deployments and resources.

   ```bash
   oc new-project wordpress
   ```
1. First, you need to create the back-end database instance, which is in our case MariaDB. From the CMD/terminal, run the following command:
   ```bash
   oc new-app mariadb-ephemeral
   ```
   
   - Note: MariaDB is not using persistent storage. So, any data stored is lost when pods are destroyed. Instead, you use a sample database in this tutorial for testing purposes.
   - Take a note or your MariaDB-generated information: MariaDB connection user name, MariaDB connection password, and MariaDB database name. For the DB host, use mariadb.

1. Because you are going to deploy WordPress, build your project on Apache with a PHP image.
   ```bash
   oc new-app php~https://github.com/wordpress/wordpress
   ```

1. expose service wordpress 
    ```bash
    oc expose svc/wordpress
    oc get routes
    ```
1. Getting the URL and Access
   - Copy the generated host name from your terminal and paste it in any browser. You should see the welcome screen of the deployed WordPress application.
   - Use the database information you previously created to complete the WordPress installation. Complete the information needed and click Install WordPress.
   - After successfully setting up WordPress, the login screen of WordPress opens. Use the user name and password set earlier and log in
   - From the WordPress dashboard, you can start building your own WordPress website.
   
1. reference:
   - https://developer.ibm.com/languages/php/tutorials/build-deploy-wordpress-on-openshift/
