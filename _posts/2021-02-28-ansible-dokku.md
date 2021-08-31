---
layout: post
title:  "Managing your dokku instance with ansible"
date:   2021-08-28 00:22:50 -0400
categories: release
tags: dokku release
---

This blog post guides you through managing your dokku instance with [ansible](https://en.wikipedia.org/wiki/Ansible_(software)).

Specifically we're going to:

  1. deploy a dokku instance
  1. provision new apps, and
  1. add users

Contrary to our last [dabble in ansible](https://dokku.github.io/general/automating-dokku-setup) from 2018, this time we're going to take advantage of the dedicated [dokku ansible role](https://github.com/dokku/ansible-dokku) that automates much of the process.

 ## Basics

[ansible](https://en.wikipedia.org/wiki/Ansible_(software)) is a tool for automating deployment, configuration management, and application management.

ansible

 * is [open-source](https://github.com/ansible/ansible), developed by [Red Hat](https://www.redhat.com/en/)
 * uses intuitive [YAML](https://docs.ansible.com/ansible/latest/reference_appendices/YAMLSyntax.html) syntax for configuration
 * is written mostly in [Python](https://www.python.org/)
 * is [agentless](https://www.ansible.com/hubfs/pdfs/Benefits-of-Agentless-WhitePaper.pdf) (if _you_ aren't running an ansible command, it is not running)
 * ships with tons of modules (bundled in [collections](https://docs.ansible.com/ansible/latest/collections/index.html)) that abstract away common tasks
 * has a whole [galaxy](https://galaxy.ansible.com/) of user-contributed components
 

In essence, ansible lets you write a couple of yaml scripts that fully characterize your dokku instance, thus treating your [infrastructure as code](https://en.wikipedia.org/wiki/Infrastructure_as_code).


## Prerequisites

### Client (your laptop, work station, ...)

You will need

- [python](https://www.python.org/)
- [git](https://git-scm.com/)
- A SSH client

Note: In the following I'm assuming the client is running a Unix OS since that's what I use. 
While ansible is certainly [designed to *manage* Windows instances](https://www.ansible.com/for/windows), I have no experience with running ansible under Windows.
Windows Subsystem for Linux should certainly work.

### Server (your future dokku instance)

At the time of writing, the `ansible-dokku` role is tested on 

 * Ubuntu 16.04 LTS, 18.04 LTS and 20.04 LTS 
 * Debian 9 and 10

You will want

 * A server running one of the operating systems listed above
 * Open ports 22 (ssh), 80 (http) and 443 (https).
 * SSH access to the server via passwordless private SSH key.
   The user must have `sudo` rights.

Further details can be found in dokku's general [installation instructions](https://dokku.com/docs/getting-started/installation).

## Getting started

In the following, we will drop the distinction between client and server because... all the commands will be run on the client.

Since you're about to begin encoding your infrastructure, it's a good idea to start a new git repository
```
$ mkdir my-dokku-playbook
$ cd my-dokku-playbook
$ git init
```

Let's create a couple of files:

`requirements.txt`: python dependencies 
```
ansible
```

`requirements.yml`: ansible role dependencies
```yaml
---
- dokku-bot.ansible-dokku
```

`ansible.cfg`: ansible configuration
```ini
[defaults]
inventory = ./hosts_development
log_path = ./ansible.log
roles_path = ./roles

# Use the YAML callback plugin.
stdout_callback = yaml
# Use the stdout_callback when running ad-hoc commands.
bin_ansible_callbacks = True
# reduce number of network operations
pipelining = True
```

`hosts_development`: ansible inventory
```ini
[dokku_servers]
dev-dokku ansible_host=148.187.96.129 ansible_user=ubuntu
```

Let's install stuff
```
$ pip install -r requirements.txt
$ mkdir ./roles
$ ansible-galaxy install -r requirements.yaml
```

And a final test to see whether ansible can connect:
```
$ ansible all -m ping
```

## Deploying the dokku instance

It's time to start "coding":

`playbook.yml`: ansible playbook for our dokku instance
```
- name: apply roles
  hosts: dokku_servers
  roles:
    # add whatever roles you need
    - role: dokku_bot.ansible_dokku
      tags: dokku
      become: true
```

`group_vars/dokku_servers.yml`: ansible variables for dokku servers
```yaml
docker_users:
  - dokku
  - ubuntu
dokku_plugins:
  - name: acl
    url: https://github.com/dokku-community/dokku-acl
```

That's it! Let's run the playbook
```
$ ansible-playbook playbook.yml
```

## Provisioning new apps

The previous section gave us a bare-bones dokku instance.
Now we want to bring it to life with our own apps, users, etc.

For the sake of brevity we are going to add the instructions to the playbook.
For increased modularity, we could just as well create a new `my_dokku` role in the `./roles` directory, bundle the new functionality in there, and then `include_role` the role in the playbook.

Let's add the following lines to our `playbook.yml`:
```yaml
- name: provision apps
  tasks:
  - name: Create seekpath app
    dokku_clone:
      app: seekpath
      repository: https://github.com/materialscloud-org/tools-seekpath
      #version: v20.09.1
    tag: my_dokku_apps

  - name: Configure ports for seekpath app
    dokku_ports:
      app: seekpath
      mappings:
        - http:80:80
        - https:443:80
      tag: my_dokku_apps
```

```shell
$ ansible-playbook playbook.yml --tags my_dokku_apps
```

## Adding users

Offering a platform as a service (PaaS) means you want to give users access to provision their own apps.
Wouldn't it be great if all you needed to do was to create a simple list of users that would automatically sync to your instance?

Let's create minimal list and work our way backward from there.

Create `host_vars/dev-dokku.yml`: ansible variables for dokku servers
```yaml
dokku_vhost_enable: 'true'

# app-level limits
my_dokku_cpu_limit: 4  # in number of CPUs
my_dokku_memory_limit: 8G # in GB

dokku_users:
  - username: ltalirz
    email: leopold.talirz@gmail.com
    ssh_key: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQChkphp/4wgTnVrYNuyE6eVNcMFzOZwglj4cP8IAtZwEde8y3KwKEHX6jG3j2tFiV9EGS+7aLVqXsHf/OFTHjad54OEafG5/BR/c2dVhk7dOGhPTCvZza8fX/9IyZiYRRBGx2lH4Y9ypSJyQ/6QzpgCe0Qd9zqnVWrqyY4ny+vsuGiSp+QWPNs0YD+tmvmPh1RvMxWdWMKGXt1QLscN3azRUl140wWLK3uAc85tqgyGCAFxOK8rOpok8GoypD2CRBNfyyislUWE+OGXxdhwOwaYEbj9vScTiMDPuEDgXpoKEaEPnMh5axmy38IXOoNhnndwwLzqiIs6ssiYUOjvmqgD"
    apps:
      - name: mofcolorizer
      - name: oximachine
      - name: colorcalibrator
      - name: mofchecker
```

The list contains
 * A `username` used by dokku internally (pick one)
 * A contact email (optional)
 * Their public SSH key
 * A list of apps they are allowed to manage

Now, in order to provision our dokku instance accordingly, we need to loop first over the list of users and then over the list of apps.
 
Let's add to our `playbook.yml`:
```yaml
- name: configure users
  - name: Create apps for dokku users
    include_tasks: user-single.yml
    vars:
      dokku_user: "{{ item }}"
    with_items: "{{ dokku_users }}"
    tags: my_dokku_users
```

`user-single.yml`: ansible task list that provisions apps for a single user
```yaml
---
- name: List apps for user "{{ dokku_user.username }}"
  command: dokku acl:allowed {{ dokku_user.username }}
  register: allowed_apps

- name: Process apps for user "{{ dokku_user.username }}"
  include_tasks: app-single.yml
  loop_control:
    loop_var: dokku_app
  with_items: "{{ dokku_user.apps }}"
```

`app-single.yml`: ansible task list that provisions a single app for a user
```yaml
---
- name: Create app {{ dokku_app.name }}
  dokku_app:
    app: "{{ dokku_app.name }}"
  when: dokku_app.name not in allowed_apps.stdout_lines

- name: Let user {{ dokku_user.username }} manage app {{ dokku_app.name}}
  command: dokku acl:add {{ dokku_app.name }} {{ dokku_user.username }}
  when: dokku_app.name not in allowed_apps.stdout_lines

- name: Set default resource limits for app {{ dokku_app.name}}
  command: >
    dokku resource:limit \
      --memory {{ my_dokku_memory_limit }} \
      --cpu {{ my_dokku_cpu_limit }} \
      {{ dokku_app.name }}
```

```
$ ansible-playbook playbook.yml --tags my_dokku_users
```


Finally, let's add some global configuration concerning which commands regular users are allowed to run
```yaml
  # see https://github.com/dokku-community/dokku-acl
  - name: Configure ACLs, set set dokku superuser "{{ my_dokku_superuser }}"
    become: true
    become_user: dokku
    lineinfile:
      path: "~dokku/.dokkurc/acl"
      line: "{{ item }}"
      create: true
    with_items:
      - export DOKKU_ACL_ALLOW_COMMAND_LINE=1
      - export DOKKU_SUPER_USER={{ my_dokku_superuser }}
      # limit commands that non-superusers can run globally
      - export DOKKU_ACL_USER_COMMANDS="help version"
      - export DOKKU_ACL_PER_APP_COMMANDS="logs urls ps:rebuild ps:restart ps:stop ps:start git-upload-pack git-upload-archive git-receive-pack git-hook storage:list proxy:ports proxy:ports-set proxy:report enter config:bundle config:clear config:export config:get config:keys config:set config:unset"
    tags: my_dokku_superuser
```


## (to decide) certificate management
...


As always, please post issues with bugs or functionality you think Dokku might benefit from. As well, feel free to hop into [Github Discussions](https://github.com/dokku/dokku/discussions) or [Slack channel](https://glider-slackin.herokuapp.com/) if you have questions, comments, or concerns.

{: .center}
[![dokku](/img/dokku.png)](http://dokku.viewdocs.io/dokku/)

---

If you're using Dokku - especially for commercial purposes - consider donating to project development via [OpenCollective](https://opencollective.com/dokku) or [Patreon](https://www.patreon.com/dokku). Funds go to general development, support, and infrastructure costs.
