# -*- mode: ruby -*-
# vi: set ft=ruby :

servers = [
    {
        :name => "k8s-master",
        :type => "master",
        :box => "centos/7",
        #:box_version => "20180831.0.0",
        :eth1 => "192.168.33.10",
	:eth2 => "192.168.0.17",
        :mem => "2048",
        :cpu => "2"
    },
    {
        :name => "k8s-node-1",
        :type => "node",
        :box => "centos/7",
        #:box_version => "20180831.0.0",
        :eth1 => "192.168.33.20",
	:eth2 => "192.168.0.18",
        :mem => "2048",
        :cpu => "2"
    }
    #{
    #    :name => "k8s-node-2",
    #    :type => "node",
    #	 :box => "centos/7",
    #    #:box => "ubuntu/xenial64",
    #    #:box_version => "20180831.0.0",
    #    :eth1 => "192.168.33.30",
    #	 :eth2 => "192.168.0.19",
    #    :mem => "2048",
    #    :cpu => "2"
    #}
]

# This script to install k8s using kubeadm && will get executed after a box is provisioned
$configureBox = <<-SCRIPT
    # Any Generic settings for Vagrant VMs
    # required for setting up password less ssh between guest VMs
    sudo sed -i "/^[^#]*PasswordAuthentication[[:space:]]no/c\PasswordAuthentication yes" /etc/ssh/sshd_config
    sudo service sshd restart

    # Establish path for Helm install
    PATH=$PATH:/usr/local/bin

    # Install docker on CentOS
    sudo -i
    yum -y update
    yum install -y docker

    # enable vagrant user to run docker commands (sudo not required)
    groupadd docker
    usermod -aG docker vagrant
    systemctl start docker && systemctl enable --now docker
    echo "####****#### DOCKER INSTALLATION COMPLETE ####****####"

    # Linux Node’s iptables to correctly see bridged traffic
    sudo modprobe br_netfilter
 
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
    sudo sysctl --system
    yum -y update

    # Disable SWAP
    sudo sed -i '/swap/d' /etc/fstab
    sudo swapoff -a
    echo "####****#### IPTABLES SETUP COMPLETE && SWAP DISABLED ####****####"
        
    # Configure Kubernetes Repository
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
    echo "####****#### KUBERNETES REPOSITORY CONFIGURED ####****####"

    # Set SELinux in permissive mode (effectively disabling it)
    setenforce 0
    sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

    # installing Kubernetes components
    #systemctl status docker
    sudo yum install -y kubelet kubeadm kubectl kubernetes-cni --disableexcludes=kubernetes
    systemctl daemon-reload && systemctl restart kubelet && systemctl enable --now kubelet
    echo "####****#### K8S COMPONENTS INSTALLED ####****####"

    # ip of this box
    #PVT_IP_ADDR=`ifconfig enp0s8 | grep Mask | awk '{print $2}'| cut -f2 -d:`
    PVT_IP_ADDR=`ip addr | grep eth1 | awk 'NR ==2 {print substr($2, 1, length($2)-3)}'`
    PUB_IP_ADDR=`ip addr | grep eth2 | awk 'NR ==2 {print substr($2, 1, length($2)-3)}'`
SCRIPT

$configureMaster = <<-SCRIPT
    # hostname & ip of this box
    HOST_NAME=$(hostname -s)
    PVT_IP_ADDR=`ip addr | grep eth1 | awk 'NR ==2 {print substr($2, 1, length($2)-3)}'`
    PUB_IP_ADDR=`ip addr | grep eth2 | awk 'NR ==2 {print substr($2, 1, length($2)-3)}'`
    echo "####****#### THIS IS MASTER NODE $HOST_NAME WITH PRIVATE IP=$PVT_IP_ADDR & PUBLIC IP=$PUB_IP_ADDR ####****####"
    
    # enable ports
    systemctl start firewalld.service
    firewall-cmd --permanent --add-port=22/tcp
    firewall-cmd --permanent --add-port=80/tcp
    firewall-cmd --permanent --add-port=443/tcp
    firewall-cmd --permanent --add-port=8000/tcp
    firewall-cmd --permanent --add-port=8080/tcp
    firewall-cmd --permanent --add-port=6443/tcp
    firewall-cmd --permanent --add-port=2379-2380/tcp
    firewall-cmd --permanent --add-port=10250-10255/tcp
    firewall-cmd --permanent --add-port=9000-9100/tcp
    sudo firewall-cmd --zone=public --add-service=http --permanent
    sudo firewall-cmd --zone=public --add-service=https --permanent
    sudo firewall-cmd --zone=public --add-service=ssh --permanent
    firewall-cmd --reload && systemctl enable firewalld.service
    #firewall-cmd --list-all
    echo "####****#### FIREWALLD, SERVICES & PORTS ENABLED ####****####"

    # install k8s master
    sudo -i
    kubeadm init --apiserver-advertise-address=$PUB_IP_ADDR --apiserver-cert-extra-sans=$PUB_IP_ADDR  --node-name $HOST_NAME --pod-network-cidr=192.168.0.0/16
    #kubeadm init --apiserver-advertise-address=$PUB_IP_ADDR --pod-network-cidr=192.168.0.0/16
    echo "####****#### KUBERNETES CLUSTER INSTALLED ####****####"

    #copying credentials to regular user - vagrant
    su - vagrant
    sudo --user=vagrant mkdir -p /home/vagrant/.kube
    cp -i /etc/kubernetes/admin.conf /home/vagrant/.kube/config
    chown $(id -u vagrant):$(id -g vagrant) /home/vagrant/.kube/config
    sudo -i

    # copying credentials to root user
    export KUBECONFIG=/etc/kubernetes/admin.conf
    #cat $KUBECONFIG && sleep 10s

    # install Calico pod network addon
    #kubectl apply -f https://docs.projectcalico.org/v3.1/getting-started/kubernetes/installation/hosted/rbac-kdd.yaml
    #kubectl apply -f https://docs.projectcalico.org/v3.1/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml
    
    # install weave pod network addon
    kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
    #kubectl get pods --all-namespaces
    kubeadm token create --print-join-command >> /etc/kubeadm_join_cmd.sh
    chmod +x /etc/kubeadm_join_cmd.sh
    cat /etc/kubeadm_join_cmd.sh && sleep 10s
SCRIPT

$configureNode = <<-SCRIPT
    # hostname & ip of this box
    HOST_NAME=$(hostname -s)
    PVT_IP_ADDR=`ip addr | grep eth1 | awk 'NR ==2 {print substr($2, 1, length($2)-3)}'`
    PUB_IP_ADDR=`ip addr | grep eth2 | awk 'NR ==2 {print substr($2, 1, length($2)-3)}'`    
    echo "####****#### THIS IS CLIENT WORKER NODE $HOST_NAME WITH PRIVATE IP=$PVT_IP_ADDR & PUBLIC IP=$PUB_IP_ADDR ####****####"
    
    # enable ports
    sudo -i
    systemctl start firewalld.service
    firewall-cmd --permanent --add-port=22/tcp
    firewall-cmd --permanent --add-port=80/tcp
    firewall-cmd --permanent --add-port=443/tcp
    firewall-cmd --permanent --add-port=10250/tcp
    firewall-cmd --permanent --add-port=30000-32767/tcp
    sudo firewall-cmd --zone=public --add-service=http --permanent
    sudo firewall-cmd --zone=public --add-service=https --permanent
    sudo firewall-cmd --zone=public --add-service=ssh --permanent
    firewall-cmd --reload && systemctl enable firewalld.service
    #firewall-cmd --list-all

    yum install -y sshpass
    sshpass -p "vagrant" scp -o StrictHostKeyChecking=no vagrant@192.168.33.10:/etc/kubeadm_join_cmd.sh .
    pwd
    cat /etc/kubeadm_join_cmd.sh
    sh ./kubeadm_join_cmd.sh
SCRIPT

$msg = <<MSG
------------------------------------------------------
	####****#### CONNECT TO THE CONTROL PLANE BY TYPING COMMAND 'vagrant ssh <server_name>' ####****####
------------------------------------------------------
MSG

Vagrant.configure("2") do |config|
    config.vm.synced_folder ".", "/vagrant"
    servers.each do |opts|
        config.vm.define opts[:name] do |config|

            config.vm.box = opts[:box]
            #config.vm.box_version = opts[:box_version]
            config.vm.hostname = opts[:name]
            config.vm.network :private_network, ip: opts[:eth1]
	    config.vm.network :public_network, ip: opts[:eth2]

            config.vm.provider "virtualbox" do |v|

                v.name = opts[:name]
                v.customize ["modifyvm", :id, "--memory", opts[:mem]]
                v.customize ["modifyvm", :id, "--cpus", opts[:cpu]]

            end

            config.vm.provision "shell", inline: $configureBox

            if opts[:type] == "master"
                config.vm.provision "shell", inline: $configureMaster
            else
                config.vm.provision "shell", inline: $configureNode
            end

        end

    end
	config.vm.post_up_message = $msg
end
