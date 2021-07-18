---
title:  "Managing Docker containers on Synology with Ansible"
categories: [ansible, docker, synology]
tags: [ansible, docker, synology]
author: Christopher Berg Schwanstrøm
---

# Managing Docker containers on Synology with Ansible

## Prerequisites

To start off we need a Linux computer on your network set up with SSH keys. On Windows it's also possible to use WSL

## Setup on the Synology

First we need to install the required applications on the Synology. We obviously need the Docker application. We also want to install a Python3 environment for Ansible to use, as we will need a few extra packages for managing Docker. The default Python3 environment does not have pip3, and we don't want to be copying packages manually.

The Docker application is in the official repos for supported units (and also possible to be installed manually on unsupported versions).

For Python3 I used a community package for this. To install community packages go to the Settings and add a package source. The URL is `http://packages.synocommunity.com`. I named it `SynoCommunity`. Then we install `Python 3.8` from here.

Then we need to create a home directory for our user to be able to use SSH authentication.

On the Synology:
```bash
mkdir -p /var/services/homes/<username>/.ssh
```

Then create the file `authorized_keys` and put your public key in there.

We also need to be able to use `sudo` without a password.

```bash
sudo su
cat <<EOF > /etc/sudoers.d/<username>
<username> ALL=(ALL) NOPASSWD:ALL
EOF
exit
```

Now we need to install the Python3.8 modules for Ansible

Provided you installed the `Python 3.8` package on volume1 we want to execute this:

```bash
/volume1/\@appstore/python38/bin/pip3 install docker
```

This will install the docker module from https://pypi.org/

## Ansible setup

Everything from here on should be run on your Linux workstation / VM / WSL / container.

Here is a very quick guide to get ansible up and running on Ubuntu (either a VM, a docker container or on WSL)

```bash
sudo apt update
sudo apt install ansible
```

## Ansible Project

This will describe a minimal Ansible project to build and deploy Docker containers

First we need to create a folder to contain the project.

```bash
mkdir -p ~/code/ansible-synology-docker
cd ~/code/ansible-synology-docker
```

The complete project will consist of these files in this folder

```
ansible.cfg
playbook.yml
hosts/synology.yml
roles/docker-nginx-example/tasks/main.yml
roles/docker-nginx-example/files/docker/Dockerfile
roles/docker-nginx-example/files/docker/index.html
```

#### ansible.cfg
This file contains defaults for the entire project and is mostly for convenience.

Here we just specify that our host definitions are located in the `hosts/` folder

```
[defaults]
inventory=hosts
```

#### hosts/synology.yml

Here we set some specific settings to the Synology. We specify the IP-address, and also the Python3 installation on the Synology from earlier. `ansible_ssh_transfer_method` is just there to disable a warning (sftp is not available on Synology, and Ansible will complain unless we specify to use scp).

We also need to specify the name of the user we want to connect as. This will be the same user we enabled passwordless sudo and ssh key authentication for.

```yml
synology_group:
  hosts:
    synology:
      ansible_host: 10.0.0.123
      ansible_user: <username>
      ansible_python_interpreter: /volume1/\@appstore/python38/bin/python3
      ansible_ssh_transfer_method: scp
```

#### roles/docker-nginx-example/files/docker/Dockerfile
```Dockerfile
FROM nginx:latest
COPY ./index.html /usr/share/nginx/html/index.html
```
#### roles/docker-nginx-example/files/docker/index.html

```html
<pre>Hello from Nginx on Docker on Synology!</pre>
```

#### roles/docker-nginx-example/tasks/main.yml

Here most of the interesting stuff happens.

The first two tasks is just creating a directory and copying the previous two files.

Then we use two of the Ansible Docker modules to first build an image, and then run the container. Here we have a lot of options, like mounting volumes for persistant storage.

https://docs.ansible.com/ansible/latest/collections/community/docker/docker_image_module.html
https://docs.ansible.com/ansible/latest/collections/community/docker/docker_container_module.html

To keep this simple, we just build our Dockerfile. Then we expose port 80 on the container on port 8765 on the host machine (the Synology). We also tell the container to restart unless it was stopped (in case we restart the Synology)

```yml
- name: Create directory for the Docker files
  file:
    name: ~/docker/nginx-test
    state: directory

- name: Copy Docker files
  copy:
    src: "{{ item }}"
    dest: ~/docker/nginx-test
  with_fileglob:
  - docker/*

- name: Build nginx container
  docker_image:
    source: build
    build:
      path: ~/docker/nginx-test
      pull: yes
    name: nginx-ansible-test-image

- name: Run nginx-certbot
  docker_container:
    name: nginx-ansible-test-container
    image: nginx-ansible-test-image
    published_ports:
    - 8765:80
    restart_policy: unless-stopped
    init: null
```


#### playbook.yml

Now to bring this together we create a playbook that simply says that the host `synology` should run the role `docker-nginx-example`.

The `become: yes` will make Ansible sudo to root and is neccessary because (as far as I have found out) there is no super simple way to allow a regular user to run docker commands directly.

```yml
 - hosts: synology
   roles:
   - docker-nginx-example
   become: yes
```

The final step is to run the playbook

```bash
ansible-playbook playbook.yml
```

Now the container should be built quite quickly, and when done it should be accessible in the browser at the url `http://10.0.0.123:8765` (if those were your actual values)