
## OKD4.7 installation - vmware
machine	OS	vCPU	RAM	Storage	Volume	IPAddress	fqdn	Remarks
bootstrap	Fedora CoreOS v33	4	16	100		167.254.227.194	okd4-bootstrap.lab.okd.local.	
master-1	Fedora CoreOS v33	4	16	100		167.254.227.195	okd4-control-plane-1.lab.okd.local.	
master-2	Fedora CoreOS v33	4	16	100		167.254.227.196	okd4-control-plane-2.lab.okd.local.	
master-3	Fedora CoreOS v33	4	16	100		167.254.227.197	okd4-control-plane-3.lab.okd.local.	
worker-1	Fedora CoreOS v33	2	8	100	200	167.254.227.198	okd4-compute-1.lab.okd.local.	
worker-2	Fedora CoreOS v33	2	8	100	200	167.254.227.199	okd4-compute-2.lab.okd.local.	
installer	Centos 8/RHEL 8.2	2	8	200		167.254.227.193	okd4-services.okd.local.						
		                        22	88	800	400			

machine	    OS	                vCPU RAM	Storage	
bootstrap	Fedora CoreOS v33	4	16	100		192.168.0.171	okd4-bootstrap.lab.okd.local.	
master-1	Fedora CoreOS v33	4	16	100		192.168.0.172	okd4-control-plane-1.lab.okd.local.	
master-2	Fedora CoreOS v33	4	16	100		192.168.0.173	okd4-control-plane-2.lab.okd.local.	
master-3	Fedora CoreOS v33	4	16	100		192.168.0.174	okd4-control-plane-3.lab.okd.local.	
worker-1	Fedora CoreOS v33	2	8	100		192.168.0.175	okd4-compute-1.lab.okd.local.	
worker-2	Fedora CoreOS v33	2	8	100		192.168.0.176	okd4-compute-2.lab.okd.local.	
installer	Centos 8/RHEL 8.2	2	8	200		192.168.0.170	okd4-services.okd.local.

- - - - - - -
## Login okd4-service VM and update OS
```
sudo dnf install -y epel-release
sudo yum install git -y
sudo yum install wget tar libvirt -y
sudo dnf update -y
```

### Setup XRDP
sudo dnf install -y xrdp tigervnc-server
sudo systemctl enable --now xrdp
sudo yum install vim-enhanced -y
sudo yum install firewalld -y
sudo systemctl enable firewalld
sudo systemctl start firewalld
sudo firewall-cmd --zone=public --permanent --add-port=3389/tcp
sudo firewall-cmd --reload

### Configure okd4-services VM to host various services (Replace with your IP address)
cd
git clone http://rtx-swtl-git.fnc.net.local/scm/cicfwk/okd-4.5.git
cd okd-4.5/okd4_files/

vim db.167.254.204
vim db.okd.local
vim haproxy.cfg
vim named.conf
vim named.conf.local
vim registry_pv.yaml

nmtui-edit ens192

### Install bind (DNS)
sudo dnf -y install bind bind-utils

### Copy the named config files and zones:
sudo cp named.conf /etc/named.conf
sudo cp named.conf.local /etc/named/
sudo mkdir /etc/named/zones
sudo cp db* /etc/named/zones

### preferably keep /etc/sysconfig/network-scripts/ifcfg-ens192 with DNS1,2,3 as follows
vi /etc/sysconfig/network-scripts/ifcfg-ens192

a. DNS1=127.0.0.1 
   DNS2=168.127.132.3
   DN3=168.127.132.3

### Enable and start named:
sudo systemctl enable named
sudo systemctl start named
sudo systemctl status named


### Create firewall rules:
sudo firewall-cmd --permanent --add-port=53/udp
sudo firewall-cmd --reload

### Test DNS on the okd4-services host ist working as expected
dig okd.local
dig -x 192.168.1.210

a. Assume 192.168.1.210 is the DNS/Services-vm ip
b. With DNS working correctly, you should see the following results:

### (Optional, this is only for multi site) Install and configure Keepalived for switchover of active and standby clusters.
# Run the following commands in Service VM of both the sites
sudo yum install keepalived -y
sudo yum install gcc kernel-headers kernel-devel -y
sudo firewall-cmd --add-rich-rule='rule protocol value="vrrp" accept' --permanent
sudo firewall-cmd --reload
#Site-A , In service VM of Active cluster, Update configuration file /etc/keepalived/keepalived.conf , replace the virtual ip with your free
floating IP choosen for virtual IP
6.
vrrp_instance VI_1 {
state MASTER
interface ens192
virtual_router_id 51
priority 255
advert_int 1
authentication {
auth_type PASS
auth_pass 1111
}
virtual_ipaddress {
192.168.27.190/24
}
}
#Site-B , In service VM of Standby cluster, Update configuration file /etc/keepalived/keepalived.conf , replace the virtual ip with your free
floating IP choosen for virtual IP
vrrp_instance VI_1 {
state BACKUP
interface ens192
virtual_router_id 51
priority 254
advert_int 1
authentication {
auth_type PASS
auth_pass 1111
}
virtual_ipaddress {
192.168.27.190/24
}
}
#Enable and start keepalived on both the sites
sudo systemctl enable keepalived
sudo systemctl start keepalived
### Check Virtual IPs
By default virtual IP will be assigned to Active server, In case of Active server gets down, it will automatically assign to the Backup server.
Use the following command to show assigned virtual IP on the interface:
ip addr show ens192
#Sample output
2: ens192: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
link/ether 00:50:56:9b:47:25 brd ff:ff:ff:ff:ff:ff
inet 192.168.27.150/24 brd 192.168.27.255 scope global noprefixroute ens192
valid_lft forever preferred_lft forever
inet 192.168.27.190/24 scope global secondary ens192
valid_lft forever preferred_lft forever

### 6. Install HAProxy
sudo dnf install haproxy -y

### Copy haproxy config from the git okd4_files directory :
sudo cp haproxy.cfg /etc/haproxy/haproxy.cfg

### Start, enable, and verify HA Proxy service:

sudo setsebool -P haproxy_connect_any 1
sudo systemctl enable haproxy
sudo systemctl start haproxy
sudo systemctl status haproxy

### Add OKD firewall ports:
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=22623/tcp
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload

### Install Apache/HTTPD
sudo dnf install -y httpd


### Change httpd to listen port to 8080:
sudo sed -i 's/Listen 80/Listen 8080/' /etc/httpd/conf/httpd.conf

### Enable and Start httpd service/Allow port 8080 on the firewall:
sudo setsebool -P httpd_read_user_content 1
sudo systemctl enable httpd
sudo systemctl start httpd
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload

### Test the webserver:
curl localhost:8080

### 8. Download the openshift-installer and oc client
#### Download the 4.7 version of the oc client and openshift-install from the OKD releases page.
cd
wget https://github.com/openshift/okd/releases/download/4.7.0-0.okd-2021-05-22-050008/openshift-client-linux-4.7.0-0.okd-2021-05-22-050008.tar.gz
wget https://github.com/openshift/okd/releases/download/4.7.0-0.okd-2021-05-22-050008/openshift-install-linux-4.7.0-0.okd-2021-05-22-050008.tar.gz

For OCP4.7 Binary Path
client wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable/openshift-client-linux.tar.gz
installer wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable/openshift-install-linux.tar.gz

### Extract the okd version of the oc client and openshift-install:
tar -zxvf openshift-client-linux-4.7.0-0.okd-2021-05-22-050008.tar.gz
tar -zxvf openshift-install-linux-4.7.0-0.okd-2021-05-22-050008.tar.gz

### Move the kubectl, oc, and openshift-install to /usr/local/bin and show the version:
sudo mv kubectl oc openshift-install /usr/local/bin/
oc version
openshift-install version

### Setup the openshift-installer:
### Generate an SSH key:
ssh-keygen
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_rsa

### Create an install directory and copy the install-config.yaml file:
cd
mkdir install_dir
cp okd-4.7/okd4_files/install-config.yaml ./install_dir


### Edit the install-config.yaml in the install_dir, insert your pull secret(cloud.redhat.com/openshift/install/pull-secret)and ssh key(~/.ssh/id_rsa.pub), and
### backup the install-config.yaml as it will be deleted in the next step:
vim ./install_dir/install-config.yaml
cp ./install_dir/install-config.yaml ./install_dir/install-config.yaml.bak

### Generate the Kubernetes manifests for the cluster, ignore the warning:
openshift-install create manifests --dir=install_dir/

### Modify the cluster-scheduler-02-config.yaml manifest file to prevent Pods from being scheduled on the control plane machines:
sed -i 's/mastersSchedulable: true/mastersSchedulable: False/' install_dir/manifests/cluster-scheduler-02-config.yml

### Create ignition-configs:
openshift-install create ignition-configs --dir=install_dir/

### Note: If you reuse the install_dir, make sure it is empty. Hidden files are created after generating the configs, and they should be removed before you use the same folder on a 2nd attempt.

### Host ignition and Fedora CoreOS files on the webserver
### Create okd4 directory in /var/www/html:
sudo mkdir /var/www/html/okd4

### Copy the install_dir contents to /var/www/html/okd4 and set permissions:
sudo cp -R install_dir/* /var/www/html/okd4/
sudo chown -R apache: /var/www/html/
sudo chmod -R 755 /var/www/html/

### Test the webserver:
curl localhost:8080/okd4/metadata.json

### Download the Fedora CoreOS bare-metal bios image and sig files and shorten the file names:
cd /var/www/html/okd4/
sudo wget https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/33.20210217.3.0/x86_64/fedora-coreos-33.20210217.3.0-metal.x86_64.raw.xz
sudo wget https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/33.20210217.3.0/x86_64/fedora-coreos-33.20210217.3.0-metal.x86_64.raw.xz.sig
For OCP4.7 Binary Path
client sudo wget
installer sudo wget
sudo mv fedora-coreos-33.20210217.3.0-metal.x86_64.raw.xz fcos.raw.xz
sudo mv fedora-coreos-33.20210217.3.0-metal.x86_64.raw.xz.sig fcos.raw.xz.sig
sudo chown -R apache: /var/www/html/
sudo chmod -R 755 /var/www/html

### Starting the Bootstrap node
### Download the Fedora CoreOS Bare Metal ISO image (https://getfedora.org/coreos/download?tab=cloud_launchable&stream=stable) and upload it in Openstack cluster. Create a new test VM using fedora live ISO image and attach it to bootstrap volume for installing configuration file.

### Power on the test VM and open VM console from openstack dashboard. Press the TAB key to edit the kernel boot options and add the following:

ip=192.168.27.170::192.168.27.1:255.255.255.0:okd4-bootstrap.lab.okd.local:ens192:none
nameserver=192.168.27.169 nameserver=168.127.132.3 nameserver=168.127.132.4 coreos.inst.install_dev=sda
coreos.inst.image_url=http://192.168.27.169:8080/okd4/fcos.raw.xz coreos.inst.ignition_url=http://192.168.27.169:8080/okd4/bootstrap.ign

### You should see that the fcos.raw.gz image and signature are downloading:
### 12. Starting the control-plane nodes
### Power on the VM and click on Console. Press the TAB key to edit the kernel boot options and add the following, then press enter:

ip=192.168.27.167::192.168.27.1:255.255.255.0:okd4-control-plane-1.lab.okd.local:ens192:none
nameserver=192.168.27.169 nameserver=168.127.132.3 nameserver=168.127.132.4 coreos.inst.install_dev=sda
coreos.inst.image_url=http://192.168.27.169:8080/okd4/fcos.raw.xz coreos.inst.ignition_url=http://192.168.27.169:8080/okd4/master.ign

### You should see that the fcos.raw.gz image and signature are downloading:

### Repeat the same process for okd4-control-plane-2 and okd4-control-plane-3 VM. Starting the compute nodes
### Power on the test VM and open VM console from openstack dashboard. Press the TAB key to edit the kernel boot options and add the following:

ip=192.168.27.168::192.168.27.1:255.255.255.0:okd4-compute-1.lab.okd.local:ens192:none
nameserver=192.168.27.169 nameserver=168.127.132.3 nameserver=168.127.132.4
coreos.inst.install_dev=sda coreos.inst.image_url=http://192.168.27.169:8080/okd4/fcos.raw.xz coreos.inst.ignition_url=http://192.168.27.169:8080/okd4/worker.ign

### You should see that the fcos.raw.gz image and signature are downloading:
### Repeat the same process for okd4-compute2 VM.
### It is usual for the worker nodes to display the following until the bootstrap process complete:

### Monitor the bootstrap installation
### You can monitor the bootstrap process from the okd4-services node:
openshift-install --dir=install_dir/ wait-for bootstrap-complete --log-level=debug

### Once the bootstrap process is complete, which can take upwards of 30 minutes, you can shutdown your bootstrap node and delete the VM. Edit the /etc/haproxy/haproxy.cfg, comment out the bootstrap node, and reload the haproxy service.
sudo sed '/ okd4-bootstrap /s/^/#/' /etc/haproxy/haproxy.cfg
sudo systemctl reload haproxy

### Login to the cluster and approve CSRs
export KUBECONFIG=~/install_dir/auth/kubeconfig
oc whoami
oc get nodes
oc get csr
cp ~/install_dir/auth/kubeconfig ~/.kube/config

### (To ssh worker/master nodes from service node: sudo ssh core@IP_address_master/worker )
### Install the jq package to assist with approving multiple CSR’s at once time.
wget -O jq https://github.com/stedolan/jq/releases/download/jq-1.6/jq-linux64
chmod +x jq
sudo mv jq /usr/local/bin/
jq --version

### Approve all the pending certs and check your nodes:
oc get csr -ojson | jq -r '.items[] | select(.status == {} ) | .metadata.name' | xargs oc adm certificate approve

### Check status of the cluster operators and cluster version.
oc get clusteroperators # all version should be true in available tab
oc get clusterversion

### Check status of nodes
oc get nodes
### Get kubeadmin password from the install_dir/auth folder and login to the web console:
cat install_dir/auth/kubeadmin-password

### Update the RDP machine /etc/hosts with below entries to access the OKD Dashboard:
192.168.27.150 console-openshift-console.apps.lab.okd.local
192.168.27.150 oauth-openshift.apps.lab.okd.local

### Open web browser to https://console-openshift-console.apps.lab.okd.local/ and login as kubeadmin with the password from above:

### 16. HTPasswd Setup:
### The kubeadmin is a temporary user. The easiest way to set up a local user is with htpasswd.
cd
cd okd-4.5/okd4_files/
htpasswd -c -B -b users.htpasswd admin admin

### Create a secret in the openshift-config project using the users.htpasswd file you generated.
oc create secret generic htpass-secret --from-file=htpasswd=users.htpasswd -n openshift-config

### Add the identity provider.
oc apply -f htpasswd_provider.yaml

### Logout of the OpenShift Console. Then select htpasswd_provider and login with admin and admin credentials.

### Check the status of installation.
openshift-install wait-for install-complete --log-level="debug"

### Procedure to add extra worker node to OKD4.7 cluster:
### Boot a Fedora CoreOS with same version used for the creating OKD4.7 cluster.
#### Start the VM and move to console tab and press the TAB key to edit the kernel boot options and add the following, then press enter:

ip=192.168.27.171::192.168.27.1:255.255.255.0:okd4-compute-2.lab.okd.local:ens192:none
nameserver=192.168.27.169 nameserver=168.127.132.3 nameserver=168.127.132.4
coreos.inst.install_dev=sda coreos.inst.image_url=http://192.168.27.169:8080/okd4/fcos.raw.xz
coreos.inst.ignition_url=http://192.168.27.169:8080/okd4/worker.ign

### Approve all the pending certs and check your nodes:
oc get csr -ojson | jq -r '.items[] | select(.status == {} ) | .metadata.name' | xargs oc adm certificate approve
***********************************************************************************************************************************************************************************************************************


## VMWare OCP 4.7 Installation Binaries:
### 1) Download the openshift-installer and oc client
### Download the RHCOS 4.7 version of the oc client and openshift-install from the OKD releases page.
cd
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable/openshift-client-linux.tar.gz
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable/openshift-install-linux.tar.gz

### Source:
https://cloud.redhat.com/openshift/install/metal/user-provisioned
https://docs.openshift.com/container-platform/4.7/installing/installing_bare_metal_ipi/ipi-install-installation-workflow.html

### Extract the okd version of the oc client and openshift-install:
tar -zxvf openshift-client-linux.tar.gz
tar -zxvf openshift-install-linux.tar.gz

### 2) Host ignition and Fedora CoreOS files on the webserver
### Create okd4 directory in /var/www/html:
sudo mkdir /var/www/html/ocp4


### Copy the install_dir contents to /var/www/html/okd4 and set permissions:
sudo cp -R install_dir/* /var/www/html/ocp4/
sudo chown -R apache: /var/www/html/
sudo chmod -R 755 /var/www/html/

### Test the webserver:
curl localhost:8080/ocp4/metadata.json

### Download the RHCOS bare-metal bios image and sig files and shorten the file names:
cd /var/www/html/ocp4/
sudo wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/latest/latest/rhcos-metal.x86_64.raw.gz
sudo mv rhcos-metal.x86_64.raw.gz rhcos.raw.gz
sudo chown -R apache: /var/www/html/
sudo chmod -R 755 /var/www/html

### Power on the bootstrap VM and open VM console from VMware dashboard. reboot the machine and Press the TAB key to edit the kernel boot options and add the following:
ip=192.168.27.164::192.168.27.1:255.255.255.0:okd4-bootstrap.lab.okd.local:ens192:none
nameserver=192.168.27.86 nameserver=168.127.132.3 nameserver=168.127.132.4
coreos.inst.install_dev=sda coreos.inst.image_url=http://192.168.27.86:8080/okd4/rhcos.raw.gz
coreos.inst.ignition_url=http://192.168.27.86:8080/okd4/bootstrap.ign
coreos.inst.insecure=yes

### Power on all the master VM's and open VM console from VMware dashboard. reboot the machine and Press the TAB key to edit the kernel boot options and add the following:
ip=192.168.27.165::192.168.27.1:255.255.255.0:okd4-control-plane-1.lab.okd.local:ens192:none
nameserver=192.168.27.86 nameserver=168.127.132.3 nameserver=168.127.132.4
coreos.inst.install_dev=sda
coreos.inst.image_url=http://192.168.27.86:8080/okd4/rhcos.raw.gz
coreos.inst.ignition_url=http://192.168.27.86:8080/okd4/master.ign
coreos.inst.insecure=yes

### Power on all the worker VM's and open VM console from VMware dashboard. reboot the machine and Press the TAB key to edit the kernel boot options and add the following:

ip=192.168.27.168::192.168.27.1:255.255.255.0:okd4-compute-1.lab.okd.local:ens192:none
nameserver=192.168.27.86 nameserver=168.127.132.3
nameserver=168.127.132.4
coreos.inst.install_dev=sda
coreos.inst.image_url=http://192.168.27.86:8080/okd4/rhcos.raw.gz
coreos.inst.ignition_url=http://192.168.27.86:8080/okd4/worker.ign
coreos.inst.insecure=yes