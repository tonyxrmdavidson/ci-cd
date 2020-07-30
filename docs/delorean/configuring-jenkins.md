# Configuring Jenkins

## Table of Contents
  - [1. Prerequisites](#prerequisites)
  - [2. Configuring the inventory](#configuring-the-inventory)
    - [2.1 Host File](#host-file)
    - [2.2 Host Vars](#host-vars)
    - [2.3 Credentials and Global Settings](#credentials-and-global-settings)
  - [3. Running the Script](#running-the-script)
    - [3.1 Configuring the Delorean Jobs](#configuring-the-delorean-jobs)
    - [3.2 Plugins Installation](#plugins-installation)
    - [3.3 Potential Issues During Configuraiton](#potential-issues-during-configuration)
  - [4. Node Configuration](#node-configuration)
    - [4.1 Tools](#tools)
    - [4.2 Red Hat Registry CA](#red-hat-registry-ca)
    - [4.3 Environment Variables](#environment-variables)
    - [4.4 SSH and Git Configuration](#ssh-and-git-configuration)


## Prerequisites
- [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#intro-installation-guide)
- [Docker](https://docs.docker.com/install/overview)


The `configure_jenkins` playbook and all of its required files for inventory and roles are located in the `scripts/` folder.

```sh
cd scripts/
```
## Configuring the inventory
The inventory files must be configured before the script can be run.

### Host File
Create the `hosts` file
```sh
cp inventories/hosts.template inventories/hosts
```

### Host Vars
A host vars file must be created for each Jenkins host. This will contain the Jenkins configuration (i.e. credentials) used to connect to the target Jenkins instance.
```sh
cp inventories/host_vars/jenkins-host.template inventories/host_vars/<jenkins-host>.yaml
```

**IMPORTANT:** The jenkins host entry under the `jenkins` group in the hosts file should match the name of this `host_vars` file. For example:

```
...
[jenkins]
jenkins.example.com
```

The host `jenkins.example.com` should have a host_vars with the filename `host_vars/jenkins.example.com.yaml`

### Credentials and Global Settings
Configure the `inventories/group_vars/all/credentials.yml` file. This contains all the configuration for credentials required by the Delorean jobs.
Configure the `inventories/group_vars/all/global_settings.yml` file. This contains all the configuration for global settings required by the Delorean jobs.

## Running the script
```sh
ansible-playbook -i inventories/hosts playbooks/configure_jenkins.yaml
```

### Configuring the Delorean Jobs
The Delorean jobs are not configured by the script by default. If you choose to configure this, set the variable `configure_delorean_jobs` to `true` as a parameter when running the script.

```sh
ansible-playbook -i inventories/hosts playbooks/configure_jenkins.yaml -e configure_delorean_jobs=true
```

A playbook is available to run this task separately

```
ansible-playbook -i inventories/hosts playbooks/configure_delorean_jobs.yaml
```

### Plugins Installation
Plugin installation can take a long time to finish. The logs for this task can be seen in the docker container running in your local machine. This is created by the ansible script and uses the image `jenkins-plugin-install`.

```sh
docker ps
docker container logs <container_id> -f
```

### Potential Issues During Configuration
Preflight checks are run before creating any resources on the target Jenkins instance.

#### Conflicting Credentials
Credentials defined in the [credentials.yml](../../scripts/inventories/group_vars/jenkins/credentials.yml) file will be created by the configuration script. Any existing credentials with the same credential ID will be overwritten. A warning will appear during the configuration process if any existing credentials are found and the user will be asked to confirm to continue the process.

#### Incompatible Plugins
The list of plugins that will be installed by the script, along with their target versions, can be found in the [plugins.txt](../../scripts/s2i/plugins.txt) file.

Any plugins that are already available in the Jenkins instance, won't be re-installed or updated. This may cause the Delorean jobs to not work as expected. A warning will appear during the configuration process if any incompatible plugins were found and the user will be asked to confirm to continue the process.

#### Plugin Dependency Errors
Dependency Errors may occur after plugin installation. This is required to be fixed manually.

If any dependency errors were found, a notification will appear during the configuration process and the user will be asked to ensure that these dependency errors are fixed and the plugins affected by these errors are properly loaded before continuing the process.

Any dependency errors that are not fixed may cause the configuration process to fail.

## Node Configuration
The Delorean jobs require a node labeled as `tester` and needs to be configured with the following tools and settings

### Tools
Tools that needs to be installed:
- ansible
- bzip2
- docker
- git
- go (Golang)
- java-1.8.0-openjdk-devel (openjdk 8)
- libselinux-python
- lsof
- ncurses-devel
- nfs-utils
- nfs-utils-lib
- nodejs (node.js 8)
- oc (openshift cli 3.11)
- python-pip
- python-wheel
- python-devel
- rpcbind
- svcat
- tar
- wget
- zlib-devel

#### Tools to be installed with pip
- ansible-tower-cli
- docker-py
- shade
- virtualenv

#### Additional Notes
##### Docker
Ensure Docker service is enabled and started

##### Go
Ensure that go is available on PATH

##### Node.js
Ensure the following packages are available globally:
- json
- gulp-cli
- gulp

### Red Hat Registry CA
Red Hat registry CA needs to be downloaded and installed. Once the CA has been downloaded, run:

```
update-ca-trust extract
```

This CA must also be saved as a user with private key credential in the Jenkins instance.

### Environment Variables
The following environment variables needs to be set and exported:
```
NODE_TLS_REJECT_UNAUTHORIZED=0
QUAY_USERNAME=<quay.io-username>
QUAY_PASSWORD=<quay.io-password>
```

### SSH and Git Configuration
Ensure the root user has a `.ssh` directory and all required `authorized_keys` and private keys are available within this directory in order to ssh into this node.

Create a `jenkins` user and add it to the group `wheel`

#### Configure sudo
The following configuration should be applied to `/etc/sudoers` 
- Ensure that users in the `wheel` group allows all command without using a password.
  ```
    %wheel	ALL=(ALL)	NOPASSWD: ALL
  ```
- Remove the line `requiretty` to disable tty
- Replace `env_reset` with `!env_reset` to ensure that env is not reset with sudo
  ```
    Defaults    !env_reset
  ```
- Ensure `secure_path` is not set by commenting out the following line:
  ```
    # Defaults    secure_path = ...
  ```

Ensure the `jenkins` user has a `.ssh` directory with `authorized_keys` available to ssh into this node as this user

#### Docker
Ensure the user `jenkins` gets added to the group `docker`