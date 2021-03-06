# Kubernetes with Vagrant and Ansible

Git clone and run `vagrant up` to start playing with your cluster.

## Table of contents

- [Files and Directories Structure](#files-and-directories-structure)
- [Code Overview](#code-overview)
- [Creates a Kubernetes cluster](#create-a-kubernetes-cluster)
  - [Container Runtime Interface](#container-runtime-interface)
  - [Kubernetes Installation](#kubernetes-installation)
  - [Configure the master](#master)
    - [Initialise the cluster](#initialise-the-cluster)
    - [Container Network Interface](#container-network-interface)   
  - [Configure the nodes](#nodes)
- [Post Installation](#post-installation)
  - [Helm](#helm)
  - [Deploy nginx web-server](#deploy-nginx-web-server)
  - [Metallb](#metallb)
  - [Ingress](#ingress)

## Files and Directories Structure
```
.
├── playbook.yml
├── roles
│  ├── cni
│  │  └── tasks
│  │     └── main.yml
│  ├── containerd
│  │  ├── defaults
│  │  │  └── main.yml
│  │  ├── handlers
│  │  │  └── main.yml
│  │  └── tasks
│  │     └── main.yml
│  ├── fish
│  │  └── tasks
│  │     └── main.yml
│  ├── kubernetes
│  │  ├── defaults
│  │  │  └── main.yml
│  │  ├── tasks
│  │  │  └── main.yml
│  ├── masters
│  │  └── tasks
│  │     └── main.yml
│  └── nodes
│     └── tasks
│        └── main.yml
└── Vagrantfile
```
## Code Overview
Vagrantfile:
```ruby
#!/usr/bin/env ruby

N = 2
NETWORK = "192.168.77"
ANSIBLE_GROUPS = {
  "masters" => ["node-1"],
  "workers" => ["node-[2:#{N}]"],
  "cluster:children" => ["masters", "workers"]
}

Vagrant.configure("2") do |config|
  config.ssh.insert_key = false
  config.vm.box = "generic/ubuntu2004"
  config.vm.provider "libvirt" do |v|
    v.memory = 2048
    v.cpus = 2
  end

  (1..N).each do |i|
    config.vm.define "node-#{i}" do |node|
      node.vm.network "private_network", ip: "#{NETWORK}.#{i + 9}"
      node.vm.hostname = "node-#{i}"
        
      if i == N
        node.vm.provision "ansible" do |ansible|
          ansible.playbook = "playbook.yml"
          ansible.compatibility_mode = "2.0"
          ansible.groups = ANSIBLE_GROUPS
          ansible.limit = "all"
        end
      end
    end
  end
end
```
The playbook:
```yaml
---
- hosts: cluster
  gather_facts: yes
  strategy: free
  become: yes
  roles:
    - fish
    - containerd
    - kubernetes

- hosts: masters
  gather_facts: yes
  become: yes
  roles:
    - masters
    - cni

- hosts: workers
  gather_facts: yes
  become: yes
  roles:
    - nodes
```

## Create a kubernetes cluster
We will be setting up a Kubernetes cluster that will consist of one master and one worker node. All the vms will run Ubuntu 20.04 and Ansible playbooks will be used for provisioning. 


### Container Runtime Interface
First of all we need to install a container runtime into each node in the cluster so that Pods can run there. We are going to use containerd as CRI. In order to have containerd configured and running we need to:

Install the package `containerd`
```yaml
- name: Install package
  apt:
    name: containerd
    state: present
    update_cache: yes
```
Create a file in `/etc/modules-load.d/containerd.conf` and add `overlay` `br_netfilter` which are kernel's module.
```yaml
- name: Configure /etc/modules-load.d/containerd.conf
  blockinfile:
    path: /etc/modules-load.d/containerd.conf
    create: yes
    block: |
      overlay
      br_netfilter
```
Add the two modules with modprobe:
```yaml
- name: Add modules
  modprobe:
    name: "{{ item }}"
    state: present
  with_items:
    - overlay
    - br_netfilter
```
Setup required sysctl params:
```yaml
- name: "Set sysctl parameters"
  sysctl:
    name: "{{ item }}"
    value: '1'
    sysctl_set: yes
    state: present
    reload: yes
  with_items:
    - net.bridge.bridge-nf-call-iptables
    - net.ipv4.ip_forward
    - net.bridge.bridge-nf-call-ip6tables
```
Create a directory in `/etc/containerd`
```yaml
- name: Create a directory
  file:
    path: /etc/containerd
    state: directory
    mode: 0755
```
Generate the default config then restart and enable containerd.
```yaml
- name: Generate default configuration
  shell: containerd config default > /etc/containerd/config.toml
  args:
    creates: /etc/containerd/config.toml
  notify: Restart containerd

- name: Enable containerd
  systemd:
    name: containerd
    state: started
    enabled: yes
```
[Container runtime](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)

**[`^        back to top        ^`](#)**

### Kubernetes Installation
Let's first install the `kubeadm` `kubelet` `kubectl`
```yaml
- name: Add Kubernetes APT GPG key
  apt_key:
    url: https://packages.cloud.google.com/apt/doc/apt-key.gpg
    state: present

- name: Add Kubernetes APT repository
  apt_repository:
    repo: deb http://apt.kubernetes.io/ kubernetes-xenial main
    state: present
    filename: 'kubernetes'

- name: Install kubernetes packages
  apt:
    name: "{{ item }}"
    update_cache: yes
    state: present
  with_items: 
    - name: kubeadm
    - name: kubectl
    - name: kubelet
```
The swap needs to be disabled:
```yaml
- name: Disable system swap
  shell: "swapoff -a"
  changed_when: false

- name: Remove current swaps from fstab
  lineinfile:
    dest: /etc/fstab
    regexp: '(?i)^([^#][\S]+\s+(none|swap)\s+swap.*)'
    line: '# \1'
    backrefs: yes
    state: present

- name: Disable swappiness and pass bridged IPv4 traffic to iptable's chains
  sysctl:
    name: "{{ item.name }}"
    value: "{{ item.value }}"
    state: present
  with_items:
    - { name: 'vm.swappiness', value: '0' }
```

[Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

[kubelet drop-in file](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/kubelet-integration/#the-kubelet-drop-in-file-for-systemd)

[Custom Management Network](https://fdio-vpp.readthedocs.io/en/latest/usecases/contiv/CUSTOM_MGMT_NETWORK.html)

**[`^        back to top        ^`](#)**

### Master

#### Initialise the cluster
The control-plane node is the machine where th control plance components run, in this example the control-plane node will be initialise with `kubeadm init <args>`
```yaml
- name: Init cluster if needed
  shell: kubeadm init --apiserver-advertise-address="{{ hostvars['node-1']['ansible_eth1']['ipv4']['address'] }}" --apiserver-cert-extra-sans="{{ hostvars['node-1']['ansible_eth1']['ipv4']['address'] }}"  --node-name master --pod-network-cidr=192.168.0.0/16
```
To make kubectl run for non-root user you need to copy the content of `/etc/kubernetes/admin.conf` to `$HOME/.kube/config`
```yaml
- name: Creates Kubernetes config directory
  when: ini_cluster is succeeded
  file:
    path: ".kube/"
    state: directory

- name: Copy admin.conf
  when: ini_cluster is succeeded
  copy:
    src: "/etc/kubernetes/admin.conf"
    dest: ".kube/config"
    owner: "{{ ansible_user }}"
    group: "{{ ansible_user }}"
    mode: 0755
    remote_src: true
```
We now need to generate a token let other nodes join the cluster
```yaml
- name: Generate join command
  command: kubeadm token create --print-join-command
  register: join_command
```

[Creating a cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

[kubeadm init](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/)

[kubeadm token](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-token/)
#### Container Network Interface
Now that kubernetes is installed we need to deploy a Container Network Interface (CNI) so that the pods can communicate with each other. 

In this example I have chosen Flannel for its simplicity:
```yaml
- name: Install Flannel Network
  become: false
  command: kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

```
[Pod Network](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network)

[Addons](https://kubernetes.io/docs/concepts/cluster-administration/addons/)

[Flannel](https://github.com/flannel-io/flannel)

[Cluster Networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/)

### Nodes
Last but not least you can add a node with the token that was stored in the previous task:
```yaml
- name: Join the node to cluster
  shell: "{{ hostvars['node-1']['join_command']['stdout'] }}"
```
**[`^        back to top        ^`](#)**

## Post Installation

### Helm
It would be nice if only kubernetes had a package manager. And It would be even better if it was easy to install:

```yml
- name: Download Helm
  get_url:
    url: https://get.helm.sh/helm-v3.8.0-linux-amd64.tar.gz
    dest: "{{ ansible_env.PWD }}"

- name: Extract archive
  unarchive:
    src: "{{ ansible_env.PWD }}/helm-v3.8.0-linux-amd64.tar.gz"
    dest: "{{ ansible_env.PWD }}"
    mode: 0755
    remote_src: yes

- name: Move binary
  copy:
    src: "{{ ansible_env.PWD }}/linux-amd64/helm"
    dest: /usr/local/bin/helm
    mode: 0755
    remote_src: yes

- name: Remove archive
  file:
    path: "{{ item }}"
    state: absent
  with_items:
    - linux-amd64
    - helm-v3.8.0-linux-amd64.tar.gz

```
If only ...

## Deploy nginx web-server

Now that we have a running Kubernetes cluster, we can deploy a containerised application on top of it. In this case it's going to be nginx.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: nginx
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.16
        ports:
        - containerPort: 80
```
In kubernetes we use services as a way to expose an application running on a set of Pods. 

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: nginx
spec:
  selector:
    app: nginx
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30200
```
Namespaces are way to provide a mechanism for isolating groups of resources within a single cluster.

In Kubernetes you can create a namespace by kubectl:
```
kubect create ns <namespace_name>
```
or with a yaml file
```yaml
---
 apiVersion: v1
 kind: Namespace
 metadata:
   name: nginx

```
If you do a `kubectl -n nginx get svc` you will see that there are no events happening that the status is stopped on `<pending>`.
```yaml
NAME            TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx-service   LoadBalancer   10.101.234.6   <pending>     80:30200/TCP   7m58s
```


[Service](https://kubernetes.io/docs/concepts/services-networking/service/)

[Namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
## Metallb
The LoadBalancer service on Kubernetes is available on virtual machines outside Cloud. This is where metallb come into play. 

In this tutorial I will be using Helm but feel fre to use another [way](https://metallb.universe.tf/installation). However be careful with the network addons, they are some restrictions depending on the [addon](https://metallb.universe.tf/installation/network-addons/). If you have followed this tutorial religiously you don't need to pay attention to this.
```yml
- name: Add Helm repository
  kubernetes.core.helm_repository:
    name: metallb
    repo_url: https://metallb.github.io/metallb

- name: Install Metallb
  kubernetes.core.helm:
    name: metallb
    chart_ref: metallb/metallb
    release_namespace: metallb
    create_namespace: true
    values: 
      configInline:
        address-pools:
        - name: default
          protocol: layer2
          addresses:
          - 192.168.1.240-192.168.1.250
```
Metallb needs some [configurations](https://metallb.universe.tf/configuration), the layer2 is the simplest to configure because it only requires Ip addresses.

After the installation the status has changed:
```sh
$ kubectl -n nginx get svc

NAME            TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)        AGE
nginx-service   LoadBalancer   10.98.229.16   192.168.1.240   80:30200/TCP   10m

```

If you do a `curl` on the external IP of the LoadBalancer we find the nginx's homepage:
```sh
$ curl 192.168.1.240
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

```
## Ingress


**[`^        back to top        ^`](#)**
