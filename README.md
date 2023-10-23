
## == DOCKER/CONTAINERD ===

Install container runtime. In this case containerd

    sudo apt-get install -y ca-certificates curl gnupg

    sudo install -m 0755 -d /etc/apt/keyrings

    curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

    sudo chmod a+r /etc/apt/keyrings/docker.gpg

    echo   "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
        "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" |   sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    
    sudo apt-get update
    
    sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

set systemd for containerd

    sudo containerd config default | sudo tee /etc/containerd/config.toml
    
    sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
    
    sudo  service containerd restart


## == KUBEADM Debian/Ubuntu flavour - Current rel. 1.28 ===

[Official docs](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

    sudo apt-get update

    sudo apt-get install -y apt-transport-https ca-certificates curl

    # Note: In releases older than Debian 12 and Ubuntu 22.04, /etc/apt/keyrings does not exist by default; you can create it by running 
    # sudo mkdir -m 755 /etc/apt/keyrings

    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

Install kubeadm / kubelet / kubectl

    sudo apt-get update
    sudo apt-get install -y kubelet kubeadm kubectl
    sudo apt-mark hold kubelet kubeadm kubectl

Generic pre-check... is kubelet systemd service masked? If so, kubelet won't start properly. For unmask it:

    systemctl unmask kubelet

Initialize k8s cluster

    sudo kubeadm init --apiserver-advertise-address 192.168.56.3 --pod-network-cidr=192.168.0.0/20

If everything went fine the output should be:

    Your Kubernetes control-plane has initialized successfully!

    To start using your cluster, you need to run the following as a regular user:

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

    Alternatively, if you are the root user, you can run:

    export KUBECONFIG=/etc/kubernetes/admin.conf

    You should now deploy a pod network to the cluster.
    Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
        https://kubernetes.io/docs/concepts/cluster-administration/addons/

    Then you can join any number of worker nodes by running the following on each as root:

        kubeadm join 192.168.56.3:6443 --token xxxxxxxxxxxxxxxx \
            --discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

## == CALICO ==

You need to install a cni manager before proceed ith further steps. In this case we're going to use Calico.

Note: if you check the pod status on control plane node, you can see that coredns pods are in Pending status.

As soon as the setup of Calico is properly completed, coredns pods will run as expected.

Install Tiger operator
    kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/tigera-operator.yaml

Install or download the Calico file
    kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/custom-resources.yaml

    # or download the file 
    # and adjust the cidr range according to your kubeadm init config
    wget https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/custom-resources.yaml

    _____________________________________________
    apiVersion: operator.tigera.io/v1
    kind: Installation
    metadata:
    name: default
    spec:
    # Configures Calico networking.
    calicoNetwork:
        # Note: The ipPools section cannot be modified post-install.
        ipPools:
        - blockSize: 26
        cidr: 192.168.0.0/16                <==============
        encapsulation: VXLANCrossSubnet
        natOutgoing: Enabled
        nodeSelector: all()
    ...
    _____________________________________________

    kubectl create -f custom-resources.yaml

Check the status on cluster

    watch kubectl get pods -n calico-system


## == FLANNEL - CKA Course ;) ==

Download the Flannel file:

     wget https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

If needed, edit it accordingly with your POD cidrs:

    net-conf.json: |
        {
        "Network": "10.244.0.0/16",
        "Backend": {
            "Type": "vxlan"
        }
        }

If you need to bind a specific network interface (for instance eth0), add to kube-flannel containers startup args:

        containers:
        - args:
            - --ip-masq
            - --kube-subnet-mgr
            - --iface=eth0          <==========

Flannel [docs](https://github.com/flannel-io/flannel/blob/master/Documentation/troubleshooting.md#vagrant) 


