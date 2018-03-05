# How to setup Kubernetes cluster using Kubeadm, Vagrant and VirtualBox in local environment.

## Disclaimer

* only tested in MacOS
* based solely on my own experience and your mileage may vary.

## Prerequisites

* Docker 
* Vagrant
* VirtualBox

## Environments
* Mac OS High Sierra 10.13.1
* Docker CE 17.12.0-cd-mac55 (23011)
* Vagrant 2.0.2
* VirtualBox 5.2.8 r121009 (Qt5.6.3)
* Kubernetes 1.9.3
* Kubeadm: 
	```
	kubeadm version: &version.Info{Major:"1", Minor:"9", GitVersion:"v1.9.3", GitCommit:"d2835416544f298c919e2ead3be3d0864b52323b", GitTreeState:"clean", BuildDate:"2018-02-07T11:55:20Z", GoVersion:"go1.9.2", Compiler:"gc", Platform:"linux/amd64"}
	```
* kubectl:
	```
	Client Version: version.Info{Major:"1", Minor:"9", GitVersion:"v1.9.3", GitCommit:"d2835416544f298c919e2ead3be3d0864b52323b", GitTreeState:"clean", BuildDate:"2018-02-07T12:22:21Z", GoVersion:"go1.9.2", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"9", GitVersion:"v1.9.3", GitCommit:"d2835416544f298c919e2ead3be3d0864b52323b", GitTreeState:"clean", BuildDate:"2018-02-07T11:55:20Z", GoVersion:"go1.9.2", Compiler:"gc", Platform:"linux/amd64"}
```


## References

* [Get Started with Kubeadm](http://docker-k8s-lab.readthedocs.io/en/latest/kubernetes/kubeadm.html)
* [Using kubeadm to Create a Cluster](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/)

## High Level Description

* The steps described here loosely follow the steps described in __Using kubeadm to Create a Cluster__
* The Vagrantfile that is used to provision Vagrant box is taken from this [one](https://github.com/xiaopeng163/docker-k8s-lab/blob/master/lab/k8s/multi-node/vagrant/Vagrantfile)
* What we are going to to is:
	* spin up 3 Vagrant boxes, one for master node and the others for worker nodes.
	* install __Docker__, __Kubeadm__, __kubelet__, __kubectl__ and __kubernetes-cni__ on each node.
	* run __kubeadm init__ to bootstrap the cluster
	* setup pod networking
	* join worker nodes with master
	* verify nodes are available


## Steps

* Step 1, go to the directory where Vagrantfile is located.
* Step 2, from command prompt, execute

	```
	vagrant up
	```
* Step 3, when step 2 finishes, execute this to ssh into the master node

	```
	vagrant ssh k8s-master
	```
	
* Step 4, execute

	```
	sudo kubeadm init --apiserver-advertise-address 192.168.205.10 --pod-network-cidr 10.244.0.0/16
	```

	Note that we are specifying __pod-network-cidr__ so that we can setup __Pod Network__ using __Flannel__.
	
	We are also specifying __apiserver-advertise-address__ with master node IP address.
	
	If everything goes well, you will see something similar to the following
	
	
	```		
	Your Kubernetes master has initialized successfully!
	
	To start using your cluster, you need to run the following as a regular user:
	
	  mkdir -p $HOME/.kube
	  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
	  sudo chown $(id -u):$(id -g) $HOME/.kube/config
	
	You should now deploy a pod network to the cluster.
	Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
	  https://kubernetes.io/docs/concepts/cluster-administration/addons/
	
	You can now join any number of machines by running the following on each node
	as root:
	
	  kubeadm join --token 70a01b.062cc83882f69723 192.168.205.10:6443 --discovery-token-ca-cert-hash sha256:121fb4e1af98415fce644dbdc8c9f7bce23b2c744050ddebdd45a7ae573c68e3
	```
	
	
* Step 5, following the instructions from the step 4's output, execute

	```
	mkdir -p $HOME/.kube
	sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
	sudo chown $(id -u):$(id -g) $HOME/.kube/config
	```
	
* Step 6, according to __Using kubeadm to Create a Cluster__, setup Pod Networking with the following instructions. Instructions are only applied to __Flannel__.

	```
	sudo sysctl net.bridge.bridge-nf-call-iptables=1
	sudo kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.9.1/Documentation/kube-flannel.yml
	
	```	
	
	If everything goes well, you will see the following output
	
	```
	clusterrole "flannel" created
	clusterrolebinding "flannel" created
	serviceaccount "flannel" created
	configmap "kube-flannel-cfg" created
	daemonset "kube-flannel-ds" created
	```


* Step 7, setup two other worker nodes. For each workder node, do 
	* open a command line terminal, execute
	
		```
		vagrant ssh k8s-worker1 (or worker2)
		```
		
	* once ssh'd into the worker node, look for the output from master node which has __kubeadm join__ command, execute it. 

		```
		sudo kubeadm join --token XXXXX 192.168.205.10:6443 --discovery-token-ca-cert-hash XXXXX
		```

* Step 8, verify nodes. From the master node, execute

	```
	sudo kubectl get nodes
	```

	If everything goes well, you will see something similar to the following
	
	```
	vagrant@k8s-master:~$ kubectl get nodes
	NAME          STATUS    ROLES     AGE       VERSION
	k8s-master    Ready     master    3m        v1.9.3
	k8s-worker1   Ready     <none>    1m        v1.9.3
	k8s-worker2   Ready     <none>    1m        v1.9.3
	```



## Teardown

* Step 1, from the master node, execute


	```
	sudo kubectl drain k8s-worker1 --delete-local-data --force --ignore-daemonsets
	sudo kubectl delete node k8s-worker1
	sudo kubectl drain k8s-worker2 --delete-local-data --force --ignore-daemonsets
	sudo kubectl delete node k8s-worker2
	sudo kubectl drain k8s-master --delete-local-data --force --ignore-daemonsets
	sudo kubectl delete node k8s-master
	sudo kubeadm reset
	```

* Step 2, from each worker node, execute

	```
	sudo kubeadm reset
	```

* Step 3, destroy Vagrant boxes. From the terminal on the host machine (not inside Vagrant box), execute


	```
	vagrant destroy
	```


## More References

* [Playing with kubeadm in Vagrant Machines](https://medium.com/@joatmon08/playing-with-kubeadm-in-vagrant-machines-36598b5e8408)



## Aftermath

Few things odd happened after the experiement

* Docker on host machine no longer able to be restarted. Have to restart the machine to get it started again.
* __kubeadm init__ command timed out. I only ran into this situation when trying it at home and had no issue when trying it in the office. Suspect something to do with networking but not able to verify.



## For the Curious

__kubeadm init__ logs

```
vagrant@k8s-master:~$ sudo kubeadm init --apiserver-advertise-address 192.168.205.10 --pod-network-cidr 10.244.0.0/16
[init] Using Kubernetes version: v1.9.3
[init] Using Authorization modes: [Node RBAC]
[preflight] Running pre-flight checks.
	[WARNING FileExisting-crictl]: crictl not found in system path
[certificates] Generated ca certificate and key.
[certificates] Generated apiserver certificate and key.
[certificates] apiserver serving cert is signed for DNS names [k8s-master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.205.10]
[certificates] Generated apiserver-kubelet-client certificate and key.
[certificates] Generated sa key and public key.
[certificates] Generated front-proxy-ca certificate and key.
[certificates] Generated front-proxy-client certificate and key.
[certificates] Valid certificates and keys now exist in "/etc/kubernetes/pki"
[kubeconfig] Wrote KubeConfig file to disk: "admin.conf"
[kubeconfig] Wrote KubeConfig file to disk: "kubelet.conf"
[kubeconfig] Wrote KubeConfig file to disk: "controller-manager.conf"
[kubeconfig] Wrote KubeConfig file to disk: "scheduler.conf"
[controlplane] Wrote Static Pod manifest for component kube-apiserver to "/etc/kubernetes/manifests/kube-apiserver.yaml"
[controlplane] Wrote Static Pod manifest for component kube-controller-manager to "/etc/kubernetes/manifests/kube-controller-manager.yaml"
[controlplane] Wrote Static Pod manifest for component kube-scheduler to "/etc/kubernetes/manifests/kube-scheduler.yaml"
[etcd] Wrote Static Pod manifest for a local etcd instance to "/etc/kubernetes/manifests/etcd.yaml"
[init] Waiting for the kubelet to boot up the control plane as Static Pods from directory "/etc/kubernetes/manifests".
[init] This might take a minute or longer if the control plane images have to be pulled.
[apiclient] All control plane components are healthy after 35.503359 seconds
[uploadconfig] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[markmaster] Will mark node k8s-master as master by adding a label and a taint
[markmaster] Master k8s-master tainted and labelled with key/value: node-role.kubernetes.io/master=""
[bootstraptoken] Using token: 050c9d.f965d883bebd4b8c
[bootstraptoken] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstraptoken] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstraptoken] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstraptoken] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: kube-dns
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join --token 050c9d.f965d883bebd4b8c 192.168.205.10:6443 --discovery-token-ca-cert-hash sha256:b6ea8da0379b7841daec00391c9313cc0b188744778c34e07a24ff1034ed3fe2
```

