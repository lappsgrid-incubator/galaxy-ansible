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

You need Python 3.8 or higher and then install the ansible and openstack packages. See  `requirements.txt` for python dependencies (ansible is a python app)

See [docs.ansible.com](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) for more details on installing ansible. Installing ansible will take a couple of minutes.

Then see [JetStream2 application-credentials](https://docs.jetstream-cloud.org/ui/cli/auth/#setting-up-application-credentials-and-openrcsh-for-the-jetstream2-cli) to obtain a script to set shell environment with openstack credentials. 

Next you have to have an open floating IP address to hook up the Galaxy web interface for public access. 

``` bash 
$ openstack floating ip list
# see if there's a floating IP (public) address that has `None` fixed IP (private subnet address))
# if none found
$ openstack floating ip create public
# public is the default network name automatically allocated by the JS2
```

Finally, you also need to get two more files `lappsgrid.crt` and `lappsgrid.key` from somebody in the LAPPS team. Those will be needed when Galaxy is added to the instance.

> **Do NOT add the keys and certificate to the repository.**

## Creating a JetStream Instance

To simplify testing there are two playbooks that can be used to provision and destroy instances on the Indiana JetStream cluster.

```bash
$ anisble-playbook jetstream.yml         # Provisions an instance on Jetstream
$ ansible-playbook jetstream-delete.yml  # Destroys the above instance
```

**WARNING**. Take great care that you do not destroy production servers with the `jetstream-delete.yml` playbook!

All parameters for the instance created are specified in `vars/jetstream.yml` file.  Edit that file to define an instance name, the ssh key if needed.  The value for the `key` (ssh key) must match a `keypair` name stored under your JS2 account (You may change the name of you local `.pem` file associated to the keypair, but you need to keep track of which file is from which keypair). Also, note that with a high chance of the galaxy instance being worked by different developers, this `pem` file should shared among in the dev team for shared access. Only the person who start the instance can specify a keypair to use under her/his account, so IT IS THE PERSON'S RESPONSIBILITY to ensure the `pem` part of the key is shared. 
You can also configure the base image name, HW specs, and security group name (firewall rules), if you want, but you probably won't. 

Now we are ready to run the Jetstream playbook:

```bash
$ ansible-playbook jetstream.yml
```

A newly provisioned  Jetstream instance will spend some time updating packages. Unless you used a fully specified base image (the `Featured-Ubuntu20` used in the example `vars/jetstream.yml` is a UNDERSPECIFIED image and [is kept up to date by the JS2 team](https://docs.jetstream-cloud.org/general/featured/)), it shouldn't take long to update, and a reboot won't be necessary after the update. But if you're using a fully versioned image, the older the image the longer this update will take. After the updates the system might need to be rebooted. Unfortunately it is not easy (or even possible?) to have Ansible perform this step so you must log in to the instance to check if the update is complete.

```bash
$ ssh -i <YOUR_PEM_FILEPATH> <USERNAME>@<IP_OF_NEW_INSTANCE>
```

* `pem` file is, of course, the file from keypair attached to the openstack instance. 
* `username` depends on the base image. For `Featured-UbuntuX` ones, it'd be `ubuntu`
* `ip` is the floating IP, now fixed to the new instance. Note that `jetstream.yml` playbook will use the first unused floating IP from the fIP list to start the instance, and update `hosts` file. Thus you can peek at the hosts file to get the IP address used for the instance. 

Run `apt upgrade -y` on the instance, if you get a file locking error the update is still in progress. If so, wait a couple of minutes and try again. If you can run the `apt upgrade` command without an error it is safe to reboot the instance with `shutdown -r now`. 

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
- Galaxy will only listen to incoming traffic to `lappsgrid_hostname` address. If you 

Edit the `hosts` file, which looks like this in the repository.

```properties
[localhost]
127.0.0.1 ansible_connection=local ansible_python_interpreter="/usr/bin/python"  # your local python path, and it should be, of course, not any python, but the python where you installed openstack and ansible packages

[galaxyservers]
jetstream ansible_host=149.165.156.140  # set automatically by the jetstream.yml playbook, don't touch.

[galaxyservers:vars]
ansible_ssh_private_key_file=~/.ssh/some-key-from-keypair.pem  # the `pem` file mentioned above
ansible_ssh_user=ubuntu  # the `username` mentioned above
ansible_python_interpreter=/usr/bin/python3  # python path in the jetstream instance. This is the almost universal python path in major linux distros, so probably don't touch
```

Then, to install lapps-galaxy in the new JS2 instance, 

```bash
$ ansible-playbook galaxy.yml
```

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


