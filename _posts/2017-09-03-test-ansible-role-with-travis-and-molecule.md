---
title: "Test Docker-related Ansible roles with Travis and Molecule"
layout: post
date: 2017-09-03 12:00:00
tag:
- ansible
- docker
- deployment
- molecule
- devops
- travis
- ci
blog: true
---

# Problem

[Molecule](http://molecule.readthedocs.io/en/latest/index.html) is probably the most simple
and effective way to test Ansible roles. By default it uses Docker containers to provide
hosts for test environments, but things get complicated if your role manipulates with Docker
containers. If using Docker driver, you'll end up with Docker-in-docker setup, which isn't
stable, complicated and won't work in many cases.

Recently, 2.0 version of Molecule was released. It introduced new driver, called `delegated`.
With that driver, Molecule skips provisioning/deprovisioning steps, allowing you
to manage instances manually. Also, you need to provide configuration, describing how
to connect to those instances. This will allow you to test against remote hosts or to
use local connection.

# Configure Molecule

Travis provisions a new virtual machine for each run, so you don't need to care about
provisioning. To make Molecule run tests into provisioned Travis VM, we need to
modify `molecule.yml` to use `delegated` driver and `local` method to connect.

```yaml
driver:
  name: delegated
  options:
    ansible_connection_options:
      connection: local
platforms:
  - name: delegated-travis-instance
```

But that's not enough, cause Ansible will still try to connect via SSH, to
change that behaviour, you need to set `ansible_connection` to `local` for
Travis VM.

```yaml
provisioner:
  name: ansible
  inventory:
    host_vars:
      delegated-travis-instance:
        ansible_connection: local
```

# Travis configuration

Here's an example of `.travis.yml`

```yaml
---
dist: trusty
sudo: required

language: python
python:
  - "2.7"

services:
  - docker

before_install:
  - deactivate
  - |
    sudo apt-key adv --recv-keys \
      --keyserver keyserver.ubuntu.com 3B4FE6ACC0B21F32 40976EAF437D05B5
  - sudo apt-get update -qq
  - sudo apt-get install -y -o Dpkg::Options::="--force-confnew" docker-ce

install:
  - sudo pip install ansible ansible-lint docker-py molecule
  - ansible --version
  - molecule --version

script:
  - molecule test
```

# Working example

Working example of setup I've described above, you can find [here](https://github.com/decayofmind/ansible-bluegreen-docker).
