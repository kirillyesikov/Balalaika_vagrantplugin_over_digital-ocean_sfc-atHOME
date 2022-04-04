# Balalaika_vagrantplugin_over_digital-ocean_sfc-atHOME

- Low RAM environment? 
- Interested in deploying an SFC with Kubernetes at home? 
- Want to experience the convenience of Cloud infrastructure provider?


# Vagrant is not up. Well it happens :). We got you :+1:

#Create a digit ocean account, create a project and lease a droplet https://www.digitalocean.com/ . Go with Ubuntu 18.04 x64 and thank us later.

#SSH into a node and let's get started

```
sudo apt update

sudo apt-get -y install python3-pip

git clone https://github.com/akraino-edge-stack/icn-nodus.git 
```
(This implementation actively uses the icn-nodus github repo https://github.com/akraino-edge-stack/icn-nodus. Eternall thank you to the authors and developers)

#Install KVM on Linux. Mind you "libvirt-bin" was split in two parts : libvirt-clients libvirt-daemon-system
(https://askubuntu.com/questions/1089753/e-package-libvirt-bin-has-no-installation-candidate)  

```
sudo apt install qemu qemu-kvm libvirt-clients libvirt-daemon-system virtinst bridge-utils

```

#Install vagrant-libvirt plugin in Linux
```
sudo apt install qemu libvirt-daemon-system libvirt-clients libxslt-dev libxml2-dev libvirt-dev zlib1g-dev ruby-dev ruby-libvirt ebtables dnsmasq-base

apt install vagrant

vagrant plugin install vagrant-libvirt
```
#Enable nested virtualazation and install the cpu-chcker
```
./node.sh
```

#setup.sh script provisions two providers for your vagrant use (Virtualbox and libvirt) .When executing the script please thank people who wrote it
#change the vagrant version in the script from 2.2.14 to the most recent one (tested with 2.2.19)
```
vagrant_version=2.2.14
if ! vagrant version &>/dev/null; then
    enable_vagrant_install=true

```
#Run the setup script selecting the libvirt provider(with a -p flag) , 
```
./setup.sh -p libvirt
```
![](https://github.com/kirillyesikov/Balalaika_vagrantplugin_over_digital-ocean_sfc-atHOME/blob/main/vagrant.png)

#Vagrantfile (change your build file to have an updated vagrant box image or to use a differenet provider if you will , here tested with libvirt)
https://app.vagrantup.com/elastic/boxes/ubuntu-18.04-x86_64 

```
box = {
  :virtualbox => { :name => 'elastic/ubuntu-18.04-x86_64', :version => '20191013.0.0'},
  :libvirt => { :name => 'intergratedcloudnative/ubuntu1804', :version => '1.0.0'}
}

```


#You can (optionally) also change the provider(-p) that will be used during the vagrant build by changing this line :
```
provider = (ENV['VAGRANT_DEFAULT_PROVIDER'] || :libvirt).to_sym

```
#Vagrantfile uses a ```/config.default.yml``` file to define the Memory and CPU count for the VM nodes(master minions servers) you are spawning
```
require 'yaml'
pdf = File.dirname(__FILE__) + '/config/default.yml'
nodes = YAML.load_file(pdf)
```
#Nano into the /config/default.yml and downgrade your deployment according to the resources avaliable to you through your Droplet

#We are ready to Vagrant up now.
```
vagrant up

```
#verify the calico-vm-sfc-setup vm installation
```
virsh console

```

#Show the list of avalaible networks (vagrant-libvirt is the one we are interested in)
```
virsh net-list
```
#Find the private ip addresses assigned to you on your vagrant-libvirt network for your SSH connection into your minion, master and tm-servers VM's
```
virsh net-dhcp-leases  vagrant-libvirt
```
#Duplicate your droplet session and SSH into the master,minons,and tms-server (Note it's easier to ssh as a "vagrant" user with the default password "vagrant" and then execute "sudo su" for root access)

# Set up a Kubernetes Cluster (Credits to @Amir Hoseein Ghorab for the help.) Below commands are executed for minons and master
```
sudo apt update
sudo ufw disable
sudo swapoff -a; sed -i '/swap/d' /etc/fstab

sudo cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system

sudo apt install -y apt-transport-https ca-certificates curl gnupg-agent software-properties-common
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt update
sudo apt-get install docker-ce docker-ce-cli containerd.io


sudo curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
sudo echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list



sudo apt update && apt install -y kubeadm=1.20.0-00 kubelet=1.20.0-00 kubectl=1.20.0-00

```
#on the Master
```
kubeadm init --kubernetes-version=1.20.0 --pod-network-cidr=10.210.0.0/16 --ignore-preflight-errors=all --apiserver-advertise-address={Your DROPLET public IP}
export KUBECONFIG=/etc/kubernetes/admin.conf
```
# The below part is taken from https://github.com/akraino-edge-stack/icn-nodus
# Eternall thank you to authors and developers
![](https://github.com/akraino-edge-stack/icn-nodus/blob/2f419e2c7573f13132f9b822df435f654acb1c0a/images/logo/nodus_logo.png)

#Ensure the master node taint for no schedule is removed and labelled with ovn4nfv-k8s-plugin=ovn-control-plane
```
nodename=$(kubectl get node -o jsonpath='{.items[0].metadata.name}')
kubectl taint node $nodename node-role.kubernetes.io/master:NoSchedule-
kubectl label --overwrite node $nodename ovn4nfv-k8s-plugin=ovn-control-plane

```
#Deploy the Calico and Multus CNI in the kubeadm master  ( https://github.com/k8snetworkplumbingwg/multus-cni     https://github.com/projectcalico/calico ))
```
kubectl apply -f deploy/calico.yaml
kubectl apply -f deploy/multus-daemonset.yaml

```
#One of major change, we required to do for calico is to enable ip forwarding in the container network namespace. This is enabled by macro allow_ip_forwarding to true in the calico cni configuration file.

```
nano calico.yaml

```

#Make sure you change the ovn_subnet and ovn_gatewayip in deploy/ovn4nfv-k8s-plugin-sfc-setup-#II.yaml. Setup Network and SubnetLenas per user configuration.

#In this example, we customize the ovn network as follows.
```
nano ovn4nfv-k8s-plugin-sfc-setup-#II.yaml

```

```

data:
  ovn_subnet: "10.154.142.0/18"
  ovn_gatewayip: "10.154.142.1/18"
  virtual-net-conf.json: |
    {
      "Network": "172.30.16.0/22",
      "SubnetLen": 24
    }

```
#Deploy the Nodus Pod network to the cluster.
```
kubectl apply -f deploy/ovn-daemonset.yaml
kubectl apply -f deploy/ovn4nfv-k8s-plugin-sfc-setup-II.yaml

```
#don't Forget to join the workers(minions) with kubeadm join command issued on your minion nodes
You can use "kubeadm token create --print-join-command" on the master to look for your join command


#Calico by default could pick the interfaces without internet access. If user required to access internet access. Run the following command to pick the interface that has internet access. (On master)
```
kubectl set env daemonset/calico-node -n kube-system IP_AUTODETECTION_METHOD=can-reach=www.google.com

```
#TM1 server

#ssh into the TM1 vm and run the following command to attach TM1 to the left provider network.
```

apt install traceroute
ip addr flush dev eth1
ip link add link eth1 name eth1.100 type vlan id 100
ip link set dev eth1.100 up
ip addr add 172.30.10.101/24 dev eth1.100
ip route del default
ip route add default via 172.30.10.3

```


#TM2 server

#ssh into the TM2 vm and run the following command to attach TM2 to the right provider network.

```
apt install traceroute
ip addr flush dev eth1
ip link add link eth1 name eth1.200 type vlan id 200
ip link set dev eth1.200 up
ip addr add 172.30.20.2/24 dev eth1.200

Run the following commands to create virtual router

ip route add 172.30.10.0/24 via 172.30.20.3
ip route add 172.30.16.0/24 via 172.30.20.3
ip route add 172.30.17.0/24 via 172.30.20.3
ip route add 172.30.18.0/24 via 172.30.20.3
ip route add 172.30.19.0/24 via 172.30.20.3

echo 1 > /proc/sys/net/ipv4/ip_forward
/sbin/iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
 iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth1.200 -o eth0 -j ACCEPT

```


#Deploy the demonstration setup
```
kubectl apply -f example/multus-net-attach-def-cr.yaml
kubectl apply -f demo/calico-nodus-secondary-sfc-setup-II/deploy/sfc-private-network.yaml
kubectl apply -f demo/calico-nodus-secondary-sfc-setup-II/deploy/slb-multiple-network.yaml
kubectl apply -f demo/calico-nodus-secondary-sfc-setup-II/deploy/ngfw.yaml
kubectl apply -f demo/calico-nodus-secondary-sfc-setup-II/deploy/sdewan-multiple-network.yaml

```

#Deploy Pods and deploy the SFCs
```
kubectl apply -f demo/calico-nodus-secondary-sfc-setup-II/deploy/namespace-right.yaml
kubectl apply -f demo/calico-nodus-secondary-sfc-setup-II/deploy/namespace-left.yaml
kubectl apply -f demo/calico-nodus-secondary-sfc-setup-II/deploy/nginx-left-deployment.yaml
kubectl apply -f demo/calico-nodus-secondary-sfc-setup-II/deploy/nginx-right-deployment.yaml
kubectl apply -f demo/calico-nodus-secondary-sfc-setup-II/deploy/sfc.yaml

```

# Refer to https://github.com/akraino-edge-stack/icn-nodus/blob/master/demo/calico-nodus-secondary-sfc-setup-II/README.md  for Various possible testing scenarios and more SFC deployments. 
