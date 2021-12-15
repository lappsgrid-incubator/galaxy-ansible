# LAPPS/Galaxy Installation

Previous versions of Galaxy were installed by cloning the Galaxy repository from GitHub and running the code.  Recent versions of Galaxy have moved away from that model and Galaxy is installed more like a traditional Python web application.  Fortunately the Galaxy Project provides an Ansible Playbook that can be used to install Galaxy.

This Anisble Playbook is based on the playbook developed in [Day 1 of the Galaxy Admin Training workshop](https://training.galaxyproject.org/training-material/topics/admin/tutorials/ansible-galaxy/tutorial.html) with the following modifications:

1. Since the Lappsgrid uses InCommon for TLS certificates the *usegalaxy_eu.certbot* role is not needed.  The location of the Lappsgrid certificate and private key is specified at the top of the `group_vars/galaxyservers.yml` file.
2. Modified the `galaxyproject.nginx` role to copy the private key (specified above) rather than having the private key embedded in the playbook.  DO NO CHECK PRIVATE KEYS INTO GITHUB!
3. Added a new role `lappsgrid.install` that copies the `datatypes_conf.xml`, `tool_conf.xml`,  tools, and datatype converters into Galaxy.  The role also changes the owner of the `/srv/galaxy/server/database` directory so it is owned by the *galaxy* user.  Not sure if this is a bug in the Galaxy playbook, but Galaxy fails to start correctly when the *galaxy* user attempts to create files in a directory owned by *root*.
4. Added the `geerlingguy.java` role to install Java.

The huge advantage of using Ansible is the Lappsgrid modifications are bundled nicely into a separate Ansible role that is included with the Galaxy playbook.

# Jetstream Instances

To simplify testing there are two playbooks that can be used to provision and destroy instances on the Indiana Jetstream cluster.  All parameters for the instance created are specfied in `vars/jetstream.yml` file.   In particular, pay attention to the IP address being used.

```
anisble-playbook jetstream.yml # Provisions an instance on Jetstream
ansible-playbook jetstream-delete.yml # Destroys the above instance
```

**NOTE** a newly provisioned  Jetstream instance will spend some time updating packages.  The older the  image is the longer this update will take. After the system updates it will need to be rebooted.  Unfortunately it is not easy (possible?) to have Ansible perform this step so you must log in to the instance to check if the update is complete.  Run `apt upgrade -y` on the instance, if you get a file locking error the update is still in progress.  If you can run the `apt upgrade` command without an error it is safe to reboot the instance.

**WARNING** Take great care that you do not destroy production servers with the `jetstream-delete.yml` playbook!

# Using the Playbook

1. Edit `group_vars/galaxyservers.yml` file and change the following variables:
   - `nginx_ssl_src_dir`: this is the location on the local machine that contains the Lappsgrid certificate and private key
   - `nginx_conf_ssl_certificate`: the name of the lappsgrid certificate file
   - `nginx_conf_ssl_certificate_key`: the name of the lappsgrid private key file
   - `lappsgrid_hostname`: the host name of the server that will be hosting the Galaxy instance.
   - `galaxy_config.galaxy.admin_users`: the email address(es) of the user accounts that will be granted admin permissions in Galaxy.  Note that the Galaxy user accounts are not created but any user that registers with one of these email addresses will be granted *admin* status.
2. Edit the `hosts` file to change the IP address of the Galaxy server and the `ansible_ssh_private_key` if needed.
3. Run the `galaxy.yml` playbook.
   `$> ansible-playbook galaxy.yml`

This playbook assumes Galaxy is the only application running on the server and puts Galaxy's directories in the root folders `/srv` and `/data`.  This can be changed by editing the variables `postgresql_backup_dir`, `galaxy_root` and `galaxy_config.galaxy.file_path` in the `group_vars/galaxyservers.yml` file.

