---
layout: post
title:  "Managing your dokku instance with ansible"
date:   2021-11-28 00:22:50 -0400
categories: release
tags: dokku release
---

This blog post guides you through managing your dokku instance with [ansible](https://en.wikipedia.org/wiki/Ansible_(software)).

Specifically we're going to:

  1. deploy a dokku instance from scratch
  1. provision new apps
  1. add users and give the access to apps

Contrary to our last [dabble in ansible from 2018](https://dokku.github.io/general/automating-dokku-setup), this time we're going to take advantage of a new, dedicated ansible role: [`ansible-dokku`](https://github.com/dokku/ansible-dokku) not only makes it a breeze to provision a new dokku instance, it also includes an array of ansible *modules* that help with managing its configuration over time.

## Basics

[ansible](https://en.wikipedia.org/wiki/Ansible_(software)) is a tool for automating deployment, configuration management, and application management.

ansible

 * is [open-source](https://github.com/ansible/ansible), developed by [Red Hat](https://www.redhat.com/en/)
 * uses intuitive [YAML](https://docs.ansible.com/ansible/latest/reference_appendices/YAMLSyntax.html) syntax for configuration
 * is written mostly in [Python](https://www.python.org/)
 * is [agentless](https://www.ansible.com/hubfs/pdfs/Benefits-of-Agentless-WhitePaper.pdf) (if _you_ aren't running an ansible command, ansible is not running)
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

**Note:** This blog post guides you through all the steps but those who don't like to copy-paste can also just get all of the code from the [accompanying git repository](https://github.com/ltalirz/ansible-playbook-dokku/) via `git clone https://github.com/ltalirz/ansible-playbook-dokku my-dokku-playbook`.

Since you're about to cast your infrastructure into code, it's a good idea to start a new git repository
```shell
mkdir my-dokku-playbook
cd my-dokku-playbook
git init
```

Let's create a couple of files:

`requirements.txt`: python dependencies 
```
ansible
```

`requirements.yml`: ansible role dependencies
```yaml
roles:
- src: dokku_bot.ansible_dokku
  version: v2021.11.28
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

`hosts_development`: ansible inventory (adapt to your server configuration)
```ini
[dokku_servers]
dev-dokku ansible_host=111.222.333.444 ansible_user=ubuntu
```

Let's install stuff
```shell
pip install -r requirements.txt
mkdir ./roles
ansible-galaxy install -r requirements.yml
```

And a final test to see whether ansible can connect:
```shell
ansible all -m ping
```

## Deploying the dokku instance

It's time to start "coding":

`playbook.yml`: ansible playbook for our dokku instance
```yaml
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
docker_users:  # users that are allowed to run docker commands
- dokku
- ubuntu
dokku_plugins:  # dokku plugins to install
- name: acl
  url: https://github.com/dokku-community/dokku-acl
- name: letsencrypt
  url: https://github.com/dokku/dokku-letsencrypt
```

That's it! Let's run the playbook
```
ansible-playbook playbook.yml
```

## Provisioning new apps

The previous section gave us a bare-bones dokku instance.
Now let's bring it to life with our own apps, users, etc.

Add the following lines to our `playbook.yml`:
```yaml
  tasks:
  - name: Create seekpath app
    dokku_clone:
      app: seekpath
      repository: https://github.com/materialscloud-org/tools-seekpath
      version: v21.06.0
      # build: true
    tags: my_dokku_apps

  - name: Configure ports for seekpath app
    dokku_ports:
      app: seekpath
      mappings:
      - http:80:80
      - https:443:80
    tags: my_dokku_apps
```

Let's run our new set of tasks without re-running the dokku role by using the `my_dokku_apps` tag:

```shell
ansible-playbook playbook.yml --tags my_dokku_apps
```

## Adding users

We want to operate the dokku platform as a service (PaaS), so we want to enable users to manage their own apps.
The dokku ansible role lets you manage users and their permissions from a central yaml file.

Let's start by creating a list with the minimum information we need, and work our way back from there.

Create `host_vars/dev-dokku.yml`: ansible variables for the `dev-dokku` server
```yaml
# app-level resource limits
my_dokku_cpu_limit: 2  # max. number of CPUs per app
my_dokku_memory_limit: 4G # max. memory per app in GBytes

dokku_users:
- username: ltalirz
  email: lt@gmail.com
  ssh_key: "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQChkphp/4wgTnVrYNuyE6eVNcMFzOZwglj4cP8IAtZwEde8y3KwKEHX6jG3j2tFiV9EGS+7aLVqXsHf/OFTHjad54OEafG5/BR/c2dVhk7dOGhPTCvZza8fX/9IyZiYRRBGx2lH4Y9ypSJyQ/6QzpgCe0Qd9zqnVWrqyY4ny+vsuGiSp+QWPNs0YD+tmvmPh1RvMxWdWMKGXt1QLscN3azRUl140wWLK3uAc85tqgyGCAFxOK8rOpok8GoypD2CRBNfyyislUWE+OGXxdhwOwaYEbj9vScTiMDPuEDgXpoKEaEPnMh5axmy38IXOoNhnndwwLzqiIs6ssiYUOjvmqgD"  # noqa
  apps:
  - name: seekpath
  - name: oximachine
  - name: mofchecker

```

The list contains, for each user:
 * A `username` used as a dokku-internal identifier (just pick one)
 * A contact `email` (optional)
 * The user's public SSH key
 * A list of `apps` the user is allowed to manage

In order to provision our dokku instance accordingly, we need to loop twice:
first over the list of users and then over the list of apps.
 
Let's add a task to our `playbook.yml`:
```yaml
- name: configure users
  - name: Create apps for dokku users
    include_tasks: user-single.yml
    vars:
      dokku_user: "{{ item }}"
    with_items: "{{ dokku_users }}"
    tags: my_dokku_users
```

Here we are looping over the users, running `app-single.yml` for each user.

`user-single.yml`: ansible task list that provisions apps for a single user
```yaml
---
- name: Process apps for user "{{ dokku_user.username }}"
  include_tasks: app-single.yml
  loop_control:
    loop_var: dokku_app
  with_items: "{{ dokku_user.apps }}"
```

And finally, for each app listed we will (1) create the app, (2) give the user access to manage the app, and (3) apply the resource limits to it.
For the moment, resource limits are global but the approach above illustrates how they could be adapted per app.

`app-single.yml`: ansible task list that provisions a single app for a user
```yaml
---
- name: Create app {{ dokku_app.name }}
  dokku_app:
    app: "{{ dokku_app.name }}"

- name: Let user {{ dokku_user.username }} manage app {{ dokku_app.name }}  # noqa 301
  dokku_acl_app:
    app: "{{ dokku_app.name }}"
    users:
    - "{{ dokku_user.username }}"

- name: Set default resource limits for app {{ dokku_app.name }}  # noqa 301
  dokku_resource_limit:
    app: "{{ dokku_app.name }}"
    resources:
      cpu: "{{ my_dokku_cpu_limit }}"
      memory: "{{ my_dokku_memory_limit }}"
```

The following will create the new apps and give the users access to them:

```
ansible-playbook playbook.yml --tags my_dokku_users
```


Finally, let's add some global configuration concerning which commands regular users are allowed to run
```yaml
  # see https://github.com/dokku-community/dokku-acl
  - name: Set global dokku configuration options
    dokku_config:
      app: '--global'
      config:
        DOKKU_ACL_ALLOW_COMMAND_LINE: "1"
        DOKKU_ACL_USER_COMMANDS: "help version"
        # 'default' is the dokku user associated with logging in via SSH
        # You can pick any other user known to dokku to be the superuser.
        # DOKKU_SUPER_USER: 'default'
        DOKKU_ACL_PER_APP_COMMANDS: "logs urls ps:rebuild ps:restart ps:stop ps:start git-upload-pack git-upload-archive git-receive-pack git-hook storage:list proxy:ports proxy:ports-set proxy:report enter config:bundle config:clear config:export config:get config:keys config:set config:unset"  # noqa 204
      tags: my_dokku_superuser
```

```
ansible-playbook playbook.yml --tags my_dokku_superuser
```

## Conclusions

That's it - you now have a dokku instance managed through ansible.

See [here](https://github.com/ltalirz/ansible-playbook-dokku) for the code used in this blog post.
Feel free to open issues on the tracker if you notice things to improve.

Thanks go to all [contributors of the `ansible-dokku` role](https://github.com/dokku/ansible-dokku/graphs/contributors).

## Further reading

#### Modularization with ansible roles

For the sake of brevity, in this post we have added all instructions directly to the `playbook.yml`.
With this approach, over time the playbook can become long and hard to read.
For increased modularity, create a new `my_dokku` ansible role in the `./roles` directory, bundle the tasks in there, and then `include_role` the role in the playbook.
See the [ansible docs on writing roles](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html) for more information.

#### Certificate management

This post hasn't touched on how to manage SSL certificates.

In order to get a SSL certificate, you'll first need a domain name (say, `dokku.me`) and point it to your dokku instance.

Once that's done, configure the domain name in your `host_vars/dev-dokku.yml`:
```yaml
dokku_vhost_enable: 'true'  # use vhost-based deployments
# Serve apps under appname.dokku.me
dokku_hostname: dokku.me
```

From here on, the [`dokku-letsencrypt` plugin](https://github.com/dokku/dokku-letsencrypt) automates the process.
Simply add the following task to your `apps-single.yml`:
```yaml
- name: configure let's encrypt certificate
  dokku_letsencrypt:
    app: myapp
  tags: my_dokku_letsencrypt
```
and add the following configuration options to your `playbook.yml` (add it _before_ the `user-single.yml` task):

```yaml
  - name: Set global dokku configuration options
    dokku_config:
      app: '--global'
      config:
        # Contact email for Let's Encrypt SSL certificates
        DOKKU_LETSENCRYPT_EMAIL: "me@provider.net"
        # Comment the line below to use the Let's Encrypt production server
        DOKKU_LETSENCRYPT_SERVER: "staging"
      tags: my_dokku_letsencrypt
```

That's all! After running
```
ansible-playbook playbook.yml --tags my_dokku_letsencrypt
```
you should be able to securely access your apps, e.g. the `seekpath` app under https://seekpath.dokku.me.
A cronjob is automatically created that takes care of renewing the certificates before they expire.

##### Wildcard certificates

The method described above is low-maintenance, but it creates a new certificate for each app.
If you are running a large dokku instance, you may want to consider creating a single wildcard certificate (e.g. for `*.dokku.me`) that can be reused by all apps.

Let's Encrypt mandates using the ACME v2 protocol for obtaining wildcard certificates, which requires you to set a DNS record both in order to acquire the certificate and for every renewal.
[Some DNS providers](https://github.com/acmesh-official/acme.sh/wiki/dnsapi) offer APIs for setting DNS records, and there are tools like [acme.sh](https://github.com/acmesh-official/acme.sh) that can talk to these APIs.
However, automating this will typically require you to permanently store an API token on your server which may pose a security risk.
One alternative is to update the wildcard certificate manually, possibly via a tagged ansible task.

---

As always, please post issues with bugs or functionality you think Dokku might benefit from. As well, feel free to hop into [Github Discussions](https://github.com/dokku/dokku/discussions) or [Slack channel](https://glider-slackin.herokuapp.com/) if you have questions, comments, or concerns.

{: .center}
[![dokku](/img/dokku.png)](http://dokku.viewdocs.io/dokku/)

---

If you're using Dokku - especially for commercial purposes - consider donating to project development via [OpenCollective](https://opencollective.com/dokku) or [Patreon](https://www.patreon.com/dokku). Funds go to general development, support, and infrastructure costs.
