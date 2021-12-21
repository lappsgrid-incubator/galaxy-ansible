# LAPPS/Galaxy Installation

Previous versions of Galaxy were installed by cloning the Galaxy repository from GitHub and running the code.  Recent versions of Galaxy have moved away from that model and Galaxy is installed more like a traditional Python web application.  Fortunately the Galaxy Project provides an Ansible Playbook that can be used to install Galaxy.

This Anisble Playbook is based on the playbook developed in [Day 1 of the Galaxy Admin Training workshop](https://training.galaxyproject.org/training-material/topics/admin/tutorials/ansible-galaxy/tutorial.html) with the following modifications:

1. Since the Lapps Grid uses InCommon for TLS certificates the *usegalaxy_eu.certbot* role is not needed.  The location of the Lapps Grid certificate and private key is specified at the top of the `group_vars/galaxyservers.yml` file.
2. Modified the `galaxyproject.nginx` role to copy the private key (specified above) rather than having the private key embedded in the playbook.  DO NO CHECK PRIVATE KEYS INTO GITHUB!
3. Added a new role `lappsgrid.install` that copies the `datatypes_conf.xml`, `tool_conf.xml`,  tools, and datatype converters into Galaxy.  The role also changes the owner of the `/srv/galaxy/server/database` directory so it is owned by the *galaxy* user.  Not sure if this is a bug in the Galaxy playbook, but Galaxy fails to start correctly when the *galaxy* user attempts to create files in a directory owned by *root*.
4. Added the `geerlingguy.java` role to install Java.

The huge advantage of using Ansible is the Lappsgrid modifications are bundled nicely into a separate Ansible role that is included with the Galaxy playbook.

The rest of this document lays out the steps to a working LAPPS/Galaxy instance on the Indiana Jetstream cluster. The two main steps are to (1) create a standard Ubuntu instance with Docker and (2) add Galaxy and LAPPS to it. Both steps are done with an Ansible playbook.

## Requirements

You need Python 3.8 or higher and then install the Ansible and Openstack packages:

```bash
$ pip install ansible
$ pip install openstacksdk
```

See [docs.ansible.com](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) for more details on installing Ansible. Installing Ansible will take a couple of minutes.

> Figuring out the environment was a bit of a hassle for me (Marc). No matter what I did Ansible kept using Python 3.7, or sometimes claimed it did and yet you could see 3.8 in the dribble. Running the JetStream playbook always gave the error "openstacksdk is required for this module". Only after I said farewell to virtual environments and installed OpenStack on both Python 3.7 and 3.8 did it work. Also, on my iMac, but not my laptop, I had to add a line `interpreter_python = /usr/local/bin/python3` to `ansible.cfg`.

<u>JetStream credentials</u>. You need access to JetStream via a file named `TG-DBS170008-openrc-v3.sh`, which has your credentials for connecting to the OpenStack cloud and getting access to the LAPS Grid project (which has the identfifier TG-DBS170008). We assume you have an account on Xsede and TACC and access to the LAPPS Grid project allocation. With that in place you can log on to [https://iu.jetstream-cloud.org/](https://iu.jetstream-cloud.org/) and go to the pulldown with your username at the top right, select "OpenStack RC File v3" and the needed file will be downloaded. Put the file at the top-level of this repository (do NOT add it to the repository). You source the file to set some environment variables:

```bash
$ source TG-DBS170008-openrc.sh
```

You will be asked you for your TACC password.

<u>IP Address</u>. The setup at some point requires you to define an available floating IP address. To get such an address you go to [https://iu.jetstream-cloud.org/](https://iu.jetstream-cloud.org/) and select "Project > Network > Floating IPs", then pick a IP address that is not mapped to a fixed IP address. You can also do this with the openstack command which is added when you installed the openstack client:

```bash
$ openstack floating ip list
```

To create a new floating IP address:

```bash
$ openstack floating ip create 4367cd20-722f-4dc2-97e8-90d98c25f12e
```

The argument is the network name or address, in this case it is the identifier of the `public` network. See [docs.openstack.org](https://docs.openstack.org/python-openstackclient/pike/cli/command-objects/floating-ip.html) for details.

<u>Keys and certificates</u>. You need a couple of keys and certificates. The key mentioned in the configuration files (see below) is `lappsgrid-shared-key.pem`. However, rumor has it that the process only works with keys created by the user itself so you may need to create your own key. At [https://iu.jetstream-cloud.org/](https://iu.jetstream-cloud.org/) select "Project > Compute > Key Pairs", save the private key locally in this repository as `jetstream.pem` (any name is fine, but it may need the `pem` extension).

>  But all that doesn't really make sense, because then only the person creating the key and the instance can get access to the instance, so some experiments are needed.

You also need to get two more files `lappsgrid.crt` and `lappsgrid.key` from somebody in the LAPPS team. Those will be needed when Galaxy is added to the instance.

Do NOT add the keys and certificate to the repository.

## Creating a JetStream Instance

To simplify testing there are two playbooks that can be used to provision and destroy instances on the Indiana Jetstream cluster.

```bash
$ anisble-playbook jetstream.yml         # Provisions an instance on Jetstream
$ ansible-playbook jetstream-delete.yml  # Destroys the above instance
```

**WARNING**. Take great care that you do not destroy production servers with the `jetstream-delete.yml` playbook!

All parameters for the instance created are specfied in `vars/jetstream.yml` file. Edit that file to define an instance name, pick an available IP address and update the key if needed. Here is what is in the version of `vars/jetstream.yml` that is commited to the repository:

```yml
---
instance_name: galaxy-admin-training
image: a08024c8-a600-4428-8d56-5efb46469589
flavor: m1.quad
key: lappsgrid-shared-key
group: galaxy
ip: 149.165.156.140
network: 588397c7-8d91-4db4-b3ea-9d293209221f
```
And here is the same file after making some needed changes and with comments added:

```yml
---
# Make sure you use a new unique instance name.
instance_name: galaxy-admin-training-3
# The id of the image named "Ubuntu 18.04 Devel and Docker v.1.35",
# does not need to be changed.
image: a08024c8-a600-4428-8d56-5efb46469589
# Using a 20GB machine with 4 CPU's 10GB of RAM, change if needed.
flavor: m1.quad
# Specify another key if needed, does not need the pem extension.
key: key-marc
# Use the galaxy security group, do not change.
group: galaxy
# Use an available floating IP address.
ip: 149.165.157.140
# This is the network named lappsgrid-network, do not change.
network: 588397c7-8d91-4db4-b3ea-9d293209221f
```

> The key file may have to be in the directory that you run ansible from, but this was not verified and it is possible that at this point we only need the name of the key and that it has to be a key defined on OpenStack.

Now we are ready to run the Jetstream playbook:

```bash
$ ansible-playbook jetstream.yml
```

Use -v for a verbose dribble or -vvv for an extremely verbose dribble. When I ran this I had no keys or certificates availabele yet, just the environment variables set by sourcing the `TG-DBS170008-openrc.sh` file.

> It is possible that this caused my problems later on, to be experimented with.

A newly provisioned  Jetstream instance will spend some time updating packages. The older the image the longer this update will take. After the updates the sytem will need to be rebooted. Unfortunately it is not easy (or even possible?) to have Ansible perform this step so you must log in to the instance to check if the update is complete. SSH in as root using the available IP address and the key:

```bash
$ ssh -i ~/.ssh/key-marc.pem root@149.165.157.140
```

Run `apt upgrade -y` on the instance, if you get a file locking error the update is still in progress. If so, wait a couple of minutes and try again. If you can run the `apt upgrade` command without an error it is safe to reboot the instance with `shutdown -r now`. At that point you will be kicked out of the remote shell, if you want to SSH back in just wait a few minutes.

After this you have an instance ready with Ubuntu and Docker, but not a trace of Galaxy and the Lapps Grid.

## Adding Galaxy/LAPPS to the instance

First edit two configuration files.

Edit `group_vars/galaxyservers.yml` file and change the following variables:

- `nginx_ssl_src_dir`: this is the location on the local machine that contains the Lappsgrid certificate and private key
- `nginx_conf_ssl_certificate`: the name of the lappsgrid certificate file
- `nginx_conf_ssl_certificate_key`: the name of the lappsgrid private key file
- `lappsgrid_hostname`: the host name of the server that will be hosting the Galaxy instance.
- `galaxy_config.galaxy.admin_users`: the email address(es) of the user accounts that will be granted admin permissions in Galaxy.  Note that the Galaxy user accounts are not created but any user that registers with one of these email addresses will be granted *admin* status.

Notes and questions:

- This is where the lappsgrid certificate and key come in (`lappsgrid.crt` and `lappsgrid.key`). Wherever you put them locally is what you use. Still no idea how they were created.
- Not sure what permissions to use on the certificate and key. I used 600.
- You will not need to change the name of the lappsgrid certificate and private key, unless new leys are generated under a different name.

Edit the `hosts` file, which looks like this in the repository.

```properties
[localhost]
127.0.0.1 ansible_connection=local ansible_python_interpreter="/usr/bin/python"

[galaxyservers]
jetstream ansible_host=149.165.156.140

[galaxyservers:vars]
ansible_ssh_private_key_file=~/.ssh/lappsgrid-shared-key.pem
ansible_ssh_user=root
ansible_python_interpreter=/usr/bin/python3
```

Use the IP address from `vars/jetstream.yml` (`149.165.157.140`) and change the key if needed (to `~/.ssh/key-marc.pem`). Note that the command `/usr/bin/python` on the instance created above is python2, it almost seems like parts of ansible reply on Python2 (see below).

In any case we can now run the playbook:

```bash
$ ansible-playbook galaxy.yml
```

In my case this ran for a couple of minutes then broke in the step where it installs Galaxy base dependencies. More precisely, it fails on this command:

```bash
$ /srv/galaxy/venv/bin/pip3 install \
    --index-url https://wheels.galaxyproject.org/simple/ \
    --extra-index-url https://pypi.python.org/simple \
    -r /srv/galaxy/server/lib/galaxy/dependencies/pinned-requirements.txt
```

The end of the stack trace is

```
    File "/tmp/pip-install-lwaxbyoq/futures_a13076a324a344b2b152c0a92ab08ba6/concurrent/futures/_base.py", line 381
      raise exception_type, self._exception, self._traceback
```

Which looks like we have a Python2 raise statement and Python3 chokes on it.

But for Keigh this worked.

This playbook assumes Galaxy is the only application running on the server and puts Galaxy's directories in the root folders `/srv` and `/data`. This can be changed by editing the variables `postgresql_backup_dir`, `galaxy_root` and `galaxy_config.galaxy.file_path` in the `group_vars/galaxyservers.yml` file.

## Adding KeyCloak

We need to add KeyCloak ([https://galaxy.ansible.com/nkinder/keycloak](https://galaxy.ansible.com/nkinder/keycloak), where do we add it? The link specifies to install it with

```bash
$ ansible-galaxy install nkinder.keycloak
```

Even after failing at running the Ansible playbook for Galaxy this seemed to succeed for me:

```
Starting galaxy role install process
- downloading role 'keycloak', owned by nkinder
- downloading role from https://github.com/nkinder/ansible-keycloak/archive/0.0.5.tar.gz
- extracting nkinder.keycloak to /Users/marc/.ansible/roles/nkinder.keycloak
- nkinder.keycloak (0.0.5) was installed successfully
```

