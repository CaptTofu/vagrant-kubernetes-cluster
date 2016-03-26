# Vagrant Kubernetes 

This is the vagrant Kubernetes environment to facilitate deployment of
Kubernetes and other components for development.

## Basic overview of Vagrant Mesos environment.

This environment uses Vagrant to create a virtual cluster and Ansible to
configure. the cluster to run Mesos and Kubernetes.

Vagrant basically does the following:

* Starts a kubernetes-master, provisions it with ansible which
  in turn installs docker, etcd, flannel, kubernetes-master (scheduler, proxy-scheduler, api), then writes configuration
  files.
  - Runs a docker registry used for internal hosts lacking outside
    access to be able to run images required by Kubernetes or other applications.
  - Runs an apt-repository container with required packages
    to install various hosts to eliminate the need for the remainder of the
    setup process to have to install large packages from remote as well as provide
    a repository for internal nodes without outside access
* Starts kubernetes-minions, provisions it with ansible which
  in turn installs docker, etcd, flannel, kubernetes-minion (kubelet), then writes configuration
  files.


## To use this environment
you must have Vagrant, Ansible (version 1.9.4) and VirtualBox installed on your host system.  Once this is done, you can do the following:

* Change directory into the genesis/vagrant-kubernetes directory.

* Install the vagrant host module.

  ```vagrant plugin install vagrant-hosts```

* Install the vagrant proxy configuration module.

  ```vagrant plugin install vagrant-proxyconf```

* Create a .vagrant directory.

  ```mkdir -p .vagrant```

* Create and edit the cluster-inventory file as needed to define the cluster
  you want to create.

  ```cp cluster-inventory.example .vagrant/cluster-inventory```

* IMPORTANT: if you want to run the docker image registry and apt repository
  on every host in your cluster, set the variable ```multiple_registry_hosts```
  in the default group in ```.vagrant/cluster-inventory```.
  Example:

  ```
  [default]
  :multiple_registry_hosts false
  ```
* IMPORTANT: if you want to use docker-registry-proxy you need to set this environment
  variable:

```
 export EXTRA_OPTS=--vault-password-file=~/.ansible/vault.txt
```

  Also change the port of docker_registry:

```group_vars/all```

```
docker_registry:
  <snip>
  port: 443 
```

* Run ```vagrant up``` to create the virtual cluster and configure it with Ansible. 

* Run ```vagrant ssh <node_name>``` to log into a node in the cluster.

* Run ```vagrant halt <node_name>``` to shutdown a node in the cluster.

* Run ```vagrant up <node_name>``` to restart or create a new node in the cluster.

* Run ```vagrant destroy <node_name>``` to completely remove a node in the cluster.
  (You will lose all of the data on the node!)

* Run ```vagrant destroy``` to completley remove the entire cluster.  (You will
  lose everything!)

* Run ```./run-ansible``` to run only the Ansible configuration scripts without
  using Vagrant.  This is much faster as long as you are not creating any new
  nodes in the cluster.  If you only want to run Ansible on a specific group in
  the cluster, run ```./run-ansible --limit <group>```.

## Help

Contact the maintainers of this project.
