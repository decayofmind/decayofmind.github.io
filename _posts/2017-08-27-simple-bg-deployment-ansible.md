---
title: "Blue-green deployments concept with Ansible and Docker containers"
layout: post
date: 2017-08-27 12:00:00
tag:
- ansible
- docker
- deployment
blog: true
---

# The idea of Blue and Green

The idea of having Blue Green deployments isn't new. It's a _nice to have_ feature
for many projects, but not much have it achieved.

Simply, the idea means, that you have two copies of your application (or even infrastructure)
running at the same time. The first one has name Green and the second one is Blue.
Also, you need to have some routing component (nginx or haproxy, for example), which
will forward requests to application only to one copy.
So, two copies are running simultaneously, but only one them is active and accepting requests.
When you deploy, you deploy to your inactive copy (environment) only. Wait until all
application instances with new version warm up, test their health and finally,
you need to switch active copy on router (modify config and reload it).
From now, requests to your application go to the freshly deployed version.

Advantages of this approach are pretty obvious:
1. Your deployments have almost no downtime. Reload of router takes less than second.
2. Rollbacks are easy. All you need is to switch active copy back to the old one.
3. You can test new version on production environment before you'll make it active.
4. With advanced router you can set certain groups of users, for which new version
will be active first.

For more info about Blue Green deployments, read [original article](http://martinfowler.com/bliki/BlueGreenDeployment.html) by Martin Fowler.

# Pure Ansible implementation

Rolling deployments with automated rollbacks are already implemented in some systems
such as [Kubernetes](https://kubernetes.io/docs/tutorials/kubernetes-basics/update-intro/).
But for small projects, complex solutions like Kubernetes are redundant and hard to implement.

So, I've implemented a logic of blue green deployments within quite simple Ansible role.
I'm using ) __Docker__ images as a distribution method for application. Mainly, cause it's
~~cool~~ probably the easiest way how to run many instances of an application on single host.
As router, I choose __nginx__, as simple and effective server for balancing web apps.

## Role's execution flow

For implementation of blue green logic is important to store states somewhere.
By states I mean:

* Color (name) of the current active environment to determine a target one.
* Current version of application deployed.
  It's needed to do not switch color if deploying the same version and to have
  Ansible run idempotent (second run with same parameters shouldn't change nothing).

Remember, the goal was to have all setup light-weight and pure Ansible, so the
idea of putting that data into some key-value store/database or service descovery
(Consul, Redis, etc.) was rejected. Instead of it, I'm making use of Ansible [local facts](http://docs.ansible.com/ansible/latest/playbooks_variables.html#local-facts-facts-d),
storing all necessary data in files in `/etc/ansible/facts.d` directory on each host.
This set of facts is called `ansible_local` and it's available during role's execution.

Here's a short description of role's execution flow:

1. Check if it's first run or not.
2. If it's not first run, retrieve from local facts current __active__ version and color.
3. Deploy application (Docker container) to __inactive__ color.
4. Check application state and health of each container by executing a simple
   health check (HTTP request).
5. If check's are OK, prepare a configuration for nginx, pointing to new containers.
6. Check nginx config file for errors.
7. Reload nginx, so requests will go to new containers.
8. Send HTTP request to nginx with application health check URL to check if routing is working.
9. Write new facts (version and active color) to local facts directory.
10. Stop (not delete) old containers if required.

For more details on role's execution, see [page on GitHub](https://github.com/decayofmind/ansible-bluegreen-docker).

## Ports limitation

Docker daemon is able to assign public ports to containers automatically although,
after daemon restart (or system reboot), your containers will get new ports from daemon,
which will break upstreams in current nginx config. To workaround that problem, there're
statically defined port ranges for each color: `402X` for blue and `407X` for green.
Taking that fact into account, we're limited to `99` instances of each color.
But it shouldn't be a problem, cause the role is suitable for small projects only.

# Setup for real world infrastructure

The role is already used on couple of small production projects.
Usually, there's a virtual IP address, shared between two or more hosts (Keepalived, VRRP protocol),
which makes hosts (Docker hosts) fault-tolerant and highly-available. Only one host is handling the virtual IP
and all requests goes to it, nginx is configured to balance load equally to all containers of active color (on all hosts).
Load of single nginx instance mustn't scare you. It needs really a ton of requests per second to
make CPU load significant.

![architecture diagram](https://github.com/decayofmind/ansible-bluegreen-docker/raw/master/docs/architecture.png)

# Links

- Project's page on GitHub (https://github.com/decayofmind/ansible-bluegreen-docker)
- Role's page on Ansible Galaxy (https://galaxy.ansible.com/decayofmind/bluegreen-docker/)
