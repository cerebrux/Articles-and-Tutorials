Automated Provisioning of DigitalOcean Cloud Servers with Salt Cloud on Ubuntu 12.04
=====

### Introduction

Salt Stack is an open-source cloud deployment, configuration management, remote execution & monitoring package. You may not have heard of [Salt Stack](http://saltstack.org), but you might be familiar with [Puppet](https://www.digitalocean.com/community/articles/how-to-install-puppet-on-a-digitalocean-vps) or [Chef](https://www.digitalocean.com/community/articles/how-to-install-chef-and-ruby-with-rvm-on-a-ubuntu-vps). Salt is a similar tool, but it is relatively lightweight as far as resources and requirements and a growing number of users consider it much easier to use.

This tutorial aims to provide the reader with a simplified, basic setup of an operable Salt Cloud master server; which DigitalOcean users can employ to automate the process of creating 1, 2 or a fleet of cloud servers.

## Infrastructure Management

Tools like Salt, Puppet and Chef allow you to issue commands on multiple machines at once; and also install and configure software. Salt has two main aspects: configuration management and remote execution; while cloud deployment is left to Salt Cloud.

## Framework

Salt Cloud is a public-cloud provisioning tool designed to automate the deployment of public-cloud servers. It integrates Salt Stack with DigitalOcean's [application programming interface (API)](https://www.digitalocean.com/api/) in a clean way; and manages cloud-based servers via virtual machine [maps](https://salt-cloud.readthedocs.org/en/latest/topics/map.html) and [profiles](http://salt-cloud.readthedocs.org/en/latest/topics/profiles.html).

#### Salt Master & Salt Minion(s)

DigitalOcean droplets (i.e. cloud servers or virtual private servers)  created with Salt Cloud are called *minions*, have Salt automatically installed, and are assigned to a specified *master*. The terms [master](http://docs.saltstack.com/topics/configuration.html#term-master) and [minion](http://docs.saltstack.com/topics/configuration.html#term-minion) refer to the controller and the controlled. The master is essentially the central coordinator for all of the minions &ndash; similar to a client/server configuration where the master is the server, and the minion is the client. Commands are executed on the minions through the master, and minions send data back to the master (unless otherwise redirected with a [returner](http://docs.saltstack.com/ref/returners/index.html)).

## Benefits

DigitalOcean droplets can be created individually or in large groups and can be provisioned and fully managed, without ever needing to be logged into. From deploying single virtual machines, to mapping and provisioning entire clouds, Salt Cloud is as scalable as you need it to be.

## Prerequisites

First, consult [How To Install Salt on Ubuntu 12.04 | DigitalOcean](https://www.digitalocean.com/community/articles/how-to-install-salt-on-ubuntu-12-04) and create a Salt master/minion.

#### Security Hardening

Any server accessible from the public Internet should be security hardened, and your Salt master is no exception:

* Change your SSH port from the default Port 22 to a random port **below 1024**, as described in **Step Five** of [Initial Server Setup with Ubuntu 12.04](https://www.digitalocean.com/community/articles/initial-server-setup-with-ubuntu-12-04);

The Salt master communicates with the minions using an AES-encrypted ZeroMQ connection. These communications are done over TCP ports 4505 and 4506, which need to be accessible on the master *only*. 

* Configure a [firewall](https://www.digitalocean.com/community/articles/how-to-setup-a-firewall-with-ufw-on-an-ubuntu-and-debian-cloud-server) and make sure to open your **custom SSH port** and **TCP Ports 4505 & 4506**;
 * The default firewall configuration tool for Ubuntu is `ufw`. To open the Salt ports, simply execute:

			sudo ufw allow [custom SSH port below 1024]/tcp
			sudo ufw allow salt
			sudo ufw enable
			sudo ufw status verbose

* Either [disable password logins](https://www.digitalocean.com/community/articles/how-to-create-ssh-keys-with-putty-to-connect-to-a-vps) or deploy [Fail2ban](https://www.digitalocean.com/community/articles/how-to-protect-ssh-with-fail2ban-on-ubuntu-12-04) & [DenyHosts](https://www.digitalocean.com/community/articles/how-to-install-denyhosts-on-ubuntu-12-04).

#### Create SSH Key Pair for DigitalOcean Control Panel

Salt Cloud uses public key encryption to secure the communication between  the Salt master and DigitalOcean. Therefore, create a directory on your master in which to store your SSH keys:

	sudo mkdir /keys

Next, execute:

	sudo ssh-keygen -t rsa

The system will respond with <code>Enter file in which to save the key (/root/.ssh/id_rsa):</code>. Copy & paste:

	/keys/digital-ocean-salt-cloud

and press <code>Enter</code>, on your keyboard. The system will next display <code>Enter passphrase (empty for no passphrase):</code>, asking you to enter an *optional* password. **Do not** enter a passphrase; and, instead, tap the <code>Enter</code> key twice.

Your new public (SSH) key is now located in <code>/keys/digital-ocean-salt-cloud.pub</code>. Finally, execute:

	cat /keys/digital-ocean-salt-cloud.pub

and copy & paste the public key into your DigitalOcean Control Panel, as outlined in **Step Three** of [How To Use SSH Keys with DigitalOcean Droplets](https://www.digitalocean.com/community/articles/how-to-use-ssh-keys-with-digitalocean-droplets) (but save the name of the public key, in your Control Panel, as <code>digital-ocean-salt-cloud.pub</code>).

#### Hostname & Fully Qualified Domain Name (FQDN)

Verify that your Salt master's hostname and FQDN are properly set. *See* [Setting the Hostname & Fully Qualified Domain Name (FQDN) on Ubuntu 12.04](https://github.com/DigitalOcean-User-Projects/Articles-and-Tutorials/blob/master/set_hostname_fqdn_on_ubuntu.md#setting-the-hostname--fully-qualified-domain-name-fqdn-on-ubuntu-server-1204-lts).

#### Acquire Installation Tools

Salt Stack is built with the Python programming language; so, we'll need <code>pip</code> (a package management system used to install and manage software packages written in Python). In addition, despite being available via Python's repositories, we'll be downloading the Salt Cloud package from Salt Stack's [GitHub repository](https://github.com/saltstack/salt-cloud/).

To install <code>pip</code> and <code>git</code> on your system, execute:

	sudo apt-get -y install python-pip git

## Install Salt Cloud Dependencies

First, execute:

	sudo apt-get -y install python-m2crypto

Then, execute:

	sudo pip install pyzmq PyYAML pycrypto msgpack-python jinja2 psutil salt

Next, execute:

	sudo pip install apache-libcloud

## Install Salt Cloud

Finally, execute:

	sudo pip install git+https://github.com/saltstack/salt-cloud.git#egg=salt_cloud

Verify Salt Cloud was successfully installed, by executing:

	salt-cloud --version

## Configure Salt Cloud

Salt Cloud runs on a module system similar to the main Salt project; and, by default, uses PyYAML syntax for its template files &ndash; but numerous other templating languages are available as well. When creating your configuration files, be sure to follow the proper formatting techniques for YAML, which involves two spaces instead of tabs. An [online YAML parser](http://yaml-online-parser.appspot.com) is available, when troubleshooting syntax issues with YAML files.

### I. Core Configuration

The core configuration of Salt Cloud is handled in the [cloud configuration file](https://salt-cloud.readthedocs.org/en/latest/topics/config.html). This file is comprised of global configurations for interfacing with various cloud providers. The default minion configuration is also set up in this file. In other words, the `cloud` file is where the minions that are created derive their configuration.

Create the cloud configuration file by executing (obviously, you can use whichever text editor you wish; but this guide assumes that you have installed the [vim text editor](https://www.digitalocean.com/community/articles/installing-and-using-the-vim-text-editor-on-a-cloud-server)):

	sudo vim /etc/salt/cloud

Now, on your keyboard, tap on the <code>i</code> key; use the arrow keys to navigate the text area; and create your `cloud` file so that it resembles the example, below (replacing <code>master.yourdomain.tld</code> with the FQDN of your Salt master):

	provider: do
	# Set the location of the Salt master
	minion:
	  master: master.yourdomain.tld

To save & exit, tap the <code>Esc</code> key, on your keyboard, followed by these keystrokes: <code>:</code>, <code>w</code>, <code>q</code>, and, finally, <code>Enter</code>.

Additional, [miscellaneous options](https://salt-cloud.readthedocs.org/en/latest/topics/misc.html) are available that can be passed in the core configuration file, if you wish.

### Cloud Provider Modules

Next, create two new directories:

	sudo mkdir /etc/salt/{cloud.profiles.d,cloud.providers.d}

These new directories will hold the DigitalOcean-specific YAML configuration files.

### II. DigitalOcean Cloud Provider Configuration

The DigitalOcean cloud provider configuration is used to control access to your DigitalOcean account. Create the [DigitalOcean cloud provider configuration](https://salt-cloud.readthedocs.org/en/latest/topics/config.html#digital-ocean) file by executing:

	sudo vim /etc/salt/cloud.providers.d/digital_ocean.conf

Tap on the <code>i</code> key; use the arrow keys on your keyboard to navigate the text area; and create your `digital_ocean.conf` file so that it resembles the example, below:

	do:
	  provider: digital_ocean
	  # Digital Ocean account keys
	  client_key: YourClientIDCopiedFromControlPanel
	  api_key: YourAPIKeyCopiedFromControlPanel
	  ssh_key_name: digital-ocean-salt-cloud.pub
	  # Directory & file name on your Salt master
	  ssh_key_file: /keys/digital-ocean-salt-cloud

To save & exit, tap the <code>Esc</code> key, on your keyboard, followed by these keystrokes: <code>:</code>, <code>w</code>, <code>q</code>, and, finally, <code>Enter</code>.

#### DigitalOcean API

Using Salt Cloud with DigitalOcean requires a <code>client\_key</code> and an <code>api\_key</code>. These can be found in the [DigitalOcean Control Panel](https://www.digitalocean.com/community/articles/the-digitalocean-control-panel), under the API Access tab.

Now, create an API key for your account by following the steps outlined in [How To Use the DigitalOcean API](https://www.digitalocean.com/community/articles/how-to-use-the-digitalocean-api). You will need to copy & paste both your DigitalOcean <code>Client ID</code> and <code>API Key</code> in the configuration file described, above.

#### Interacting with the API

After you configure the DigitalOcean provider in <code>/etc/salt/cloud.providers.d/digital_ocean.conf</code>, you gain access to the following commands:

	sudo salt-cloud --list-images do
	sudo salt-cloud --list-sizes do
	sudo salt-cloud --list-locations do
	sudo salt-cloud --help

The output from these commands are important because it provides the variables needed to build our virtual machine profiles.

### III. DigitalOcean Cloud Profile

Create the DigitalOcean cloud [profiles](https://salt-cloud.readthedocs.org/en/latest/topics/profiles.html) for your server fleet, by executing:

	sudo vim /etc/salt/cloud.profiles.d/digital_ocean.conf

Now, on your keyboard, tap on the <code>i</code> key; use the arrow keys to navigate the text area; and create your `digital_ocean.conf` file so that it resembles the example, below:

	# Official distro images available for Arch, CentOS, Debian, Fedora, Ubuntu

	ubuntu_512MB_ny2:
	  provider: do
	  image: Ubuntu 12.04 x64
	  size: 512MB
	#  script: Optional Deploy Script Argument
	  location: New York 2
	  private_networking: True

	ubuntu_1GB_ny2:
	  provider: do
	  image: Ubuntu 12.04 x64
	  size: 1GB
	#  script: Optional Deploy Script Argument
	  location: New York 2
	  private_networking: True

	ubuntu_2GB_ny2:
	  provider: do
	  image: Ubuntu 12.04 x64
	  size: 2GB
	#  script: Optional Deploy Script Argument
	  location: New York 2
	  private_networking: True

	# Create additional profiles, if you wish
	#[profile_alias_of_your_choosing]:
	#  provider: do
	#  image: [from salt-cloud --list-images do]
	#  size: [from salt-cloud --list-sizes do]
	#  script: [optional deployment script e.g. Ubuntu, Fedora, python-bootstrap, etc.]
	#  location: [from salt-cloud --list-locations do]
	#  private_networking: [True or False: currently only available in NY2 region]

To save & exit, tap the <code>Esc</code> key, on your keyboard, followed by these keystrokes: <code>:</code>, <code>w</code>, <code>q</code>, and, finally, <code>Enter</code>.

In addition to the examples provided, Salt Cloud can accommodate [multiple configuration files](https://salt-cloud.readthedocs.org/en/latest/topics/profiles.html#multiple-configuration-files), which allows for more extensible configuration and plays nicely with various configuration management tools as well as version control systems.

#### OS Support for Cloud VMs

Salt Cloud works primarily by executing a [script](https://salt-cloud.readthedocs.org/en/latest/topics/deploy.html) on the newly-provisioned droplets as soon as they become available. By default, the script that is used is the [salt-bootstrap script](https://github.com/saltstack/salt-bootstrap), unless a different deploy script is declared in the cloud profile. The preferred method (as of Salt Cloud v0.8.9) is currently to use the default `salt-bootstrap` script. If the `salt-bootstrap` script does not meet your needs, you may pass [Deploy Script Arguments](https://salt-cloud.readthedocs.org/en/latest/topics/misc.html#deploy-script-arguments) or [write your own](https://salt-cloud.readthedocs.org/en/latest/topics/deploy.html).

#### Advanced Topic

A number of options exist when creating virtual machines that are beyond the scope of this article. After you feel comfortable with the foundational principles outlined in this tutorial, you may want to learn about creating a more complex setup with a [map file](https://salt-cloud.readthedocs.org/en/latest/topics/map.html). The map file allows for a number of virtual machines to be created and associated with specific profiles.

## Provision a New Cloud Server!

To create a new cloud server, execute (replacing <code>hostname</code> with any hostname of your choice):

	sudo salt-cloud --profile ubuntu_512MB_ny2 hostname

If all goes well, you should have a newly-provisioned server, bootstrapped with Salt minion (the new minion's SSH keys will automatically be added to the Salt master). If you would like to provision multiple virtual machines from the same profile, you can do so with a single command, e.g.

	sudo salt-cloud -p ubuntu_1GB_ny2 hostname1 hostname2 hostname3

(Note that <code>--profile</code> and <code>-p</code> are interchangable.)

## Minion Configuration

To configure your new fleet of cloud servers, consult: [How To Create Your First Salt Formula | DigitalOcean](https://www.digitalocean.com/community/articles/how-to-create-your-first-salt-formula).

## Destroy a Minion

There are various [options](https://salt-cloud.readthedocs.org/en/latest/ref/cli/salt-cloud.html#options) that can be passed when executing a Salt Cloud command. For example, to destroy a particular minion, simply execute:

	sudo salt-cloud -d hostname

## Additional Resources

* [Salt Cloud Documentation](https://salt-cloud.readthedocs.org/en/latest/index.html);
* [Salt Starters](http://saltstarters.org/) (a place to find and share Salt-Stack-related code, such as state trees or custom modules);
* [Salt Stack Formulas | GitHub](https://github.com/saltstack-formulas);
* [Frequently Asked Questions | Salt Stack](http://docs.saltstack.com/faq.html);
* All DigitalOcean [Configuration Management](https://www.digitalocean.com/community/community_tags/configuration-management) articles.

As always, if you need help with the steps outlined in this How-To, look to the DigitalOcean Community for assistance by posing your question(s), below.

<p><div style="text-align: right; font-size:smaller;">Article submitted by: <a href="https://plus.google.com/107285164064863645881?rel=author" target="_blank">Pablo Carranza</a> &bull; October 25, 2013</div></p>