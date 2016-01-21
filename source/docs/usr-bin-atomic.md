## Atomic: /usr/bin/atomic

The [`atomic` command](https://github.com/projectatomic/atomic) (`/usr/bin/atomic`) defines the entrypoint for Project Atomic hosts. 

On an Atomic host, you have two main software delivery vehicles:

 * rpm-ostree for managing the deployment and updates of the host system.
 * Linux containers (currently Docker) to provide containers running services and applications. 

The goal of `atomic` is to provide a high-level, coherent entrypoint for the system and to fill in gaps in Linux container implementations.

### Note about Atomic

The `atomic` command is still a bit new and undergoing rapid development. It is currently in the Fedora and CentOS Atomic host builds.

## Using /usr/bin/atomic

### `atomic run`

Atomic allows an image provider to specify how a container image expects to be run.

Specifically this includes the privilege level required.

For example if you built an 'ntpd' container application, that required the
SYS_TIME capability, you could add meta data to your container image using the
command:

```
LABEL RUN /usr/bin/docker run -d --cap-add=SYS_TYPE ntpd
```

Now if you executed `atomic run ntpd`, it would read the `LABEL RUN` json metadata from the container image and execute this command.

### `atomic install`

Most of the time when you ship an application, you need to run an install script.  This script would configure the system to run the application, for example it might configure a systemd unit file or configure kubernetes to run the application.  

This tool will allow application developers to embed the install and uninstall scripts within the application.  The application developers can then define the `LABEL INSTALL` and `LABEL UNINSTALL` methods, in the image metadata.  Here is a simple httpd installation description.

```
# Example Dockerfile for httpd application
#
FROM 		fedora
MAINTAINER	Dan Walsh
ENV container docker
RUN yum -y update; yum -y install httpd; yum clean all

LABEL Vendor="Red Hat" License=GPLv2
LABEL Version=1.0
LABEL INSTALL="docker run --rm --privileged -v /:/host -e HOST=/host -e LOGDIR=${LOGDIR} -e CONFDIR=${CONFDIR} -e DATADIR=${DATADIR} -e IMAGE=IMAGE -e NAME=NAME IMAGE /bin/install.sh"
LABEL UNINSTALL="docker run --rm --privileged -v /:/host -e HOST=/host -e IMAGE=IMAGE -e NAME=NAME IMAGE /bin/uninstall.sh"
ADD root /

EXPOSE 80

CMD [ "/usr/sbin/httpd", "-D", "FOREGROUND" ]
```

Here, `atomic install` will read the `LABEL INSTALL` line and substitute `NAME` with the name specified with the name option, or use the image name, it will also replace `IMAGE` with the image name.  It will also create the following directories:

* CONFDIR=/etc/NAME
* LOGDIR=/var/log/NAME
* DATADIR=/var/lib/NAME

To be used by the application.  The install script could populate these directories if necessary.

In this example the INSTALL method will execute the `install.sh` script, which we add to the image. The root sub-directory contains the following scripts:

```
cat root/usr/bin/install.sh
```

```
#!/bin/sh
# Make Data Dirs
mkdir -p ${HOST}/${CONFDIR} ${HOST}/${LOGDIR}/httpd ${HOST}/${DATADIR}

# Copy Config
cp -pR /etc/httpd ${HOST}/${CONFDIR}

# Create Container
chroot ${HOST} /usr/bin/docker create -v /var/log/${NAME}/httpd:/var/log/httpd:Z -v /var/lib/${NAME}:/var/lib/httpd:Z --name ${NAME} ${IMAGE}

# Install systemd unit file for running container
sed -e "s/TEMPLATE/${NAME}/g" etc/systemd/system/httpd_template.service > ${HOST}/etc/systemd/system/httpd_${NAME}.service

# Enabled systemd unit file
chroot ${HOST} /usr/bin/systemctl enable /etc/systemd/system/httpd_${NAME}.service
```

### `atomic uninstall`

The `atomic unistall` command does the same variable substitution as described for install, and can be used to remove any host system configuration.

Here is the example script we used.

```
cat root/usr/bin/uninstall.sh 
```

```
#!/bin/sh
chroot ${HOST} /usr/bin/systemctl disable /etc/systemd/system/httpd_${NAME}.service
rm -f ${HOST}/etc/systemd/system/httpd_${NAME}.service
```

Finally here is the systemd unit file template we used:

```
cat root/etc/systemd/system/httpd_template.service 
```

```
# cat ./root/etc/systemd/system/httpd_template.service 
[Unit]
Description=The Apache HTTP Server for TEMPLATE
After=docker.service

[Service]
ExecStart=/usr/bin/docker start TEMPLATE
ExecStop=/usr/bin/docker stop TEMPLATE
ExecReload=/usr/bin/docker exec -t TEMPLATE /usr/sbin/httpd $OPTIONS -k graceful

[Install]
WantedBy=multi-user.target
```
