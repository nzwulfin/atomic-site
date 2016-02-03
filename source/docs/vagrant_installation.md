# Project Atomic Vagrant Guide

## About this Guide
Project Atomic provides a platform to deploy and manage containers on bare-metal, virtual, or cloud-based servers.  Atomic hosts are designed to be minimal hosts focused on the delivery of container services.  This guide focusing on getting a standalone Atomic host up and available to run Docker containers in the quickest way possible.

### Objectives of the guide
You will configure:

* Vagrant to start an Atomic host

At the end of this guide, you will have:

* an Atomic host capable of running Docker containers

### Used in this guide
|  |  |
|---|---|
| Platform Host OS | Fedora 23 Workstation |
| Virtualization | Vagrant with libvirt provider |
| Atomic Host OS | Fedora 23 Atomic v 23.45 |

## Installing with vagrant

* If you are familiar with Vagrant, you can skip ahead to the [advanced Vagrantfile setup](#advanced-vagrant-configuration) section of the document.  

* If you haven't worked with Vagrant much, keep reading to walk through the steps to get the box and launch your first Atomic host.

The [Fedora](https://atlas.hashicorp.com/fedora/boxes/23-atomic-host) and [CentOS](https://atlas.hashicorp.com/centos/boxes/atomic-host) projects publish Vagrant boxes on Atlas for different virtualization platforms.This guide was written using the Fedora box for libvirt/KVM.

### Set up the environment

Vagrant boxes are created and managed on a per directory basis, so create a new working directory on your workstation to store the local vagrant config for the Atomic host.  The configuration file and the cache for this virtual machine will reside in the working directory.

```
mkdir ~/vagrant
cd ~/vagrant
```

### Simple Vagrant configuration

The instructions for creating a box live in a Vagrantfile.  The Vagrantfile we're using here is extremely simple, just enough to get the machine running.  The name of box in Atlas based on account and identifier the project used to publish the box.

If you'd like to see more options, take a look at the [Vagrant docs](http://docs.vagrantup.com/v2/vagrantfile/).

```
# -*- mode: ruby -*-
Vagrant.configure(2) do |config|
  config.vm.box = "fedora/23-atomic-host"
end
```

### Advanced Vagrant configuration
Vagrantfile configurations are very powerful.  The following example will create 4 Fedora 23 Atomic hosts, configure the virtual machines, and create an additional private network in libvirt.  

You can not only control the creation of the virtual machine, you can aslo control the SSH and other operations of Vagrant for a particular set of hosts.  Here we're using the same ssh key for all of the virtual machines to simplify access.  Vagrant will use the ssh agent keyring for authentication, so you can add the Vagrant default key with ssh-add.

```
ssh-add -k ~/.vagrant.d/insecure_private_key
```

```
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "fedora/23-atomic-host"
  config.ssh.forward_agent = true
  config.ssh.insert_key = false

  (1..4).each do |i|
    config.vm.define vm_name = "f23-atomic-#{i}" do |host|
      host.vm.hostname = vm_name
      host.vm.provider :libvirt do |domain|
        domain.memory = 2048
        domain.cpus = 2
      end #provider info
      ip = "172.21.12.#{i+100}"
      host.vm.network :private_network, :libvirt__network_name => "storage", :ip => ip
    end #host info
  end #each loop
end #config info

```

We maintain a [complex Vagrantfile](http://pa.io/has/an/example/somewhere) that you can use for your projects.

### Launch the Atomic host
Once you have either of the Vagrantfiles in your local directory, we're ready to launch our Atomic host.  Still in the working directory, we can run and connect to the Atomic host.  

To start the Atomic host, run:

`vagrant up`

To log into the Atomic host, run:

`vagrant ssh`

Vagrant will have created a user `vagrant` with an ssh key and sudo privileges on the system.  If you exit the session normally, the virtual machine continues running.  

```
# virsh list
 Id    Name                           State
----------------------------------------------------
 3     vagrant_default                running
```

You can reconnect with `vagrant ssh`, stop the VM with `vagrant halt`, or terminate the VM with `vagrant destroy`.

## Next steps
With an Atomic host up and running as a virtual machine, let's take a look at a few typical things we'd do.  Or head over to the [Getting Started Guide](http://www.projectatomic.io/docs/gettingstarted/) and set up a new cluster.

### Update your Atomic host
First of all, it's a good practice to always update your Atomic host.  The nature of updates via `rpm-ostree` means that you can easily roll back from OS updates should a problem be discovered.

As the vagrant user:

```
[vagrant@localhost ~]$ sudo atomic host upgrade
Updating from: fedora-atomic:fedora-atomic/f23/x86_64/docker-host

735 metadata, 3275 content objects fetched; 164499 KiB transferred in 113 seconds
Copying /etc changes: 22 modified, 0 removed, 43 added
Transaction complete; bootconfig swap: yes deployment count change: 1
```

Reboot once the new tree is in place for the upgrade to complete.

```
sudo systemctl reboot
```

### Run a Docker container
Once the virtual machine has rebooted, reconnect with `vagrant ssh`.  Atomic hosts are all about Docker so let's run a quick test here too.  We don't have any local containers or base images, so docker will go out to the Hub, find the Fedora base image, and then run `/bin/echo "Hello World!"` inside the container.

```
[vagrant@localhost ~]$ sudo docker run fedora /bin/echo "Hello World!"
Unable to find image 'fedora:latest' locally
latest: Pulling from docker.io/fedora
48ecf305d2cf: Pull complete
ded7cd95e059: Pull complete
Digest: sha256:02aaf336456b9eca88d22b2c85cfea14457d53994f53fa6360585a6c0c74e490
Status: Downloaded newer image for docker.io/fedora:latest
Hello World!
```
