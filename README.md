# natefanaro/k8s-base

This is meant to help me (and hopefully you) to set up and operate a bare metal on-prem kubernetes cluster.

Heavily influenced by this project: [DIY Kubernetes cluster with x86 stick-pcs](https://hackernoon.com/diy-kubernetes-cluster-with-x86-stick-pcs-b0b6b879f8a7)

# Hardware

* 3x [UP board](https://up-board.org/up/specifications/)
* 3x [IBERLS AC to DC Regulated Transformer Wall Power Adapter Supply Cord Plug Charger 5V 3A(3000mA) for LED Pixel Light, USB-HUB, Kindle Fire Tablet, DJ Controller, Nextbook](https://amzn.to/2Y0RIv6)
* [TP-Link 8-Port Gigabit Ethernet Switch TL-SG108E](https://amzn.to/2Y5YdfU)
* [Cable Matters 10-Pack Snagless Cat6 Ethernet Cable (Cat6 Cable/Cat 6 Cable) in Black 1 Foot - Available 1FT - 14FT in Length](https://amzn.to/2Y5YiAe)
* [CLOUDLET CASE: For Raspberry Pi and Other Single Board Computers](https://amzn.to/2Fl0psV)
* 8Gb or larger USB thumb drive for installing the OS

# Installing Ubuntu 18.10

#### Download

* [Ubuntu Server](https://www.ubuntu.com/download/server)
* [balenaEtcher](https://www.balena.io/etcher) 

#### Installation Notes

Use the standard install process for all three UP boards. Download [Ubuntu Server](https://www.ubuntu.com/download/server), use [balenaEtcher](https://www.balena.io/etcher) to write this image on a flash drive, and then boot each machine with the drive to install Ubuntu.

Each node should be named with the pattern `node0x` (eg: node01, node02, node03, etc.)

Use the same username and password for each host. Be sure that during setup each install is using all of the available drive space on the device. 

There is no need to install any additional software during the setup process.

# Setting up the cluster

At this point you should have three (or more) machines that you can SSH in to. To make the next steps easier I use [iTerm 2](https://iterm2.com) and split a new window three ways. Start a new SSH command to each IP address of the cluster and use iTerm's broadcast input feature. This will make each keystroke go to all three machines at once.

    sudo -i
    apt update && apt upgrade

Visit [Install Docker CE](https://docs.docker.com/v17.09/engine/installation/linux/docker-ce/ubuntu/#install-docker-ce) and follow the instructions. Once you run `sudo apt-get install docker-ce` you are done with that page.

The next steps to follow were found in this blog post: [Your instant Kubernetes cluster by Alex Ellis](https://blog.alexellis.io/your-instant-kubernetes-cluster/) Shortened list of instructions are here just in case that link dies.

    sudo apt-get update \
      && sudo apt-get install -y apt-transport-https \
      && curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

    echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" \
      | sudo tee -a /etc/apt/sources.list.d/kubernetes.list \
      && sudo apt-get update

    sudo apt-get update && sudo apt-get install -y kubelet kubeadm kubernetes-cni

    sudo kubeadm init

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

    kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"

    # Run the kubeadm join command that kubeadm init gave you after setting up weave
    # The command looks something like this
    sudo kubeadm join --token <token> <node01 ip>:6443 --discovery-token-ca-cert-hash sha256:<hash>

Now you should be done. If you run `kubectl get nodes` on the first node you can see a list of all three nodes. It may take a minute for them to all be in a ready state.

# Accessing the cluster

Copy `.kube/config` to your local development machine. You will also want to install `kubectl` to access the cluster.

# Networking

If you can, set up your router to always give the same static ip address to these three machines. On my network I also created a new vlan and ensured that both my home network and the new vlan could talk to each other.

Next you should set up [MetalLB](https://metallb.universe.tf). According to their page...

> MetalLB hooks into your Kubernetes cluster, and provides a network load-balancer implementation. In short, it allows you to create Kubernetes services of type “LoadBalancer” in clusters that don’t run on a cloud provider, and thus cannot simply hook into paid products to provide load-balancers.

I followed this tutorial [MetalLB - Layer 2 mode tutorial](https://metallb.universe.tf/tutorial/layer2/). Now when you create a LoadBalancer, it will receive a new IP address accessible on your network. This is magic to me as to how it works, so your milage may vary. Once this is set up you only need to set up a normal LoadBalancer service to generate a new IP address.

#### Quick Install

First, edit `metallb-config.yaml` and set up an IP range on your network. These IP addresses will be used when you create a new LoadBalancer.

Then run this to install and configure MetalLB.

    kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.7.3/manifests/metallb.yaml
    kubectl apply -f metallb-config.yaml


# Set up kubernetes-dashboard

While not required, I like to be sure that kubernetes-dashboard can be set up on a cluster. It is a good test to be sure most things are working. (For personal reasons, helm will not be used)

Visit [https://github.com/kubernetes/dashboard](https://github.com/kubernetes/dashboard) for a complete set of steps, help, support, etc.

To begin, this will set up a loadbalancer, the pod, and a few other things to get the dashboard running.

	kubectl apply -f dashboard.yaml

If you are unsure about making a user to access this, also apply `dashboard-user.yaml`

	kubectl apply -f dashboard-user.yaml

##### Getting the url

You can get the IP address of the service with this command

	kubectl get services --namespace=kube-system kubernetes-dashboard

... and use the external IP to see the dashboard.

Example:

	❰nate❙~/git/k8s-base(git≠master)❱✔≻ kubectl get services --namespace=kube-system kubernetes-dashboard
	NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)         AGE
	kubernetes-dashboard   LoadBalancer   10.98.42.140   192.168.2.200   443:32595/TCP   15m

Visit [https://192.168.2.200](https://192.168.2.200). You may have to approve your browser to access this url since there is no valid cert.

##### Bearer token

This page will require a bearer token for authentication. The `dashboard-user.yaml` file created this for us. To retrieve it...

	kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
	
Copy the "token" and paste it in to the dashboard page for access.

# Troubleshooting the install

Get the status of your nodes and all pods.

    kubectl get nodes && kubectl get pods --all-namespaces

Look for errors starting kubernetes cluster. Useful if a node does not get in to Ready state.

    journalctl -xeu kubelet

If you encounter an error and want to try reinstalling kubernetes, these commands will reset everything to where you can run `kubeadm init` again.

    kubeadm reset
    iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
    docker system prune -a --volumes
    sudo apt-get remove -y kubelet kubeadm kubernetes-cni && sudo apt autoremove
    sudo rm -rf /opt/cni/bin
    sudo rm -rf $HOME/.kube/config
    sudo rm -rf /etc/kubernetes

Resizing volume to max capacity

    sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv

# Extras

I ended up making an account with [weave](https://weave.works) and installed their agent. Do this on node01 (or from your dev machine) This gets you basic alerts and some idea of what your cluster is doing at any given time.

    curl -Ls https://get.weave.works | sh -s -- --token=<your token here>
