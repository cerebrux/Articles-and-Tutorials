# ~~~~~~~~~~ DRAFT ~~~~~~~~~~
*[Pull Requests](https://github.com/DigitalOcean-User-Projects/Articles-and-Tutorials/pulls) gladly accepted* 
How To Install & Configure SOGo - an Open-Source Alternative to Microsoft Exchange - on Ubuntu Server
=====

### Introduction

SOGo is a free and modern scalable groupware server. It offers shared calendars, address books, and emails through your favorite Web browser and by using a native client such as Mozilla Thunderbird and Lightning. In addition, SOGo offers native Microsoft Outlook compatibility using the [OpenChange](http://www.openchange.org/) backend. This means that  Microsoft Outlook 2003, 2007 and 2010 can talk directly to SOGo &ndash; just like if it was a Microsoft Exchange server; without the need for additional plugins required to make this work.

## SOGo Features

* Scalable architecture suitable for deployments from dozens to many thousands of users
* Rich Web-based interface that shares the look and feel, the features and the data of Mozilla 
Thunderbird and Lightning
* Improved integration with Mozilla Thunderbird and Lightning by using the SOGo Connector and the SOGo Integrator
* Two-way synchronization support with any SyncML-capable devices (BlackBerry, Palm, Windows CE, etc.) by using the Funambol SOGo Connector

SOGo is standard-compliant. It supports CalDAV, CardDAV, GroupDAV, iMIP and iTIP and reuses existing IMAP, SMTP and database servers &ndash; making the solution easy to deploy and interoperable with many applications. Mobile devices supporting the SyncML standard use the Funambol middleware to synchronize information.

## Prerequisites

SOGo reuses many components in an infrastructure. Thus, it requires the following:

* Database server (e.g. [MySQL](https://www.digitalocean.com/community/community_tags/mysql) or [PostgreSQL](https://www.digitalocean.com/community/community_tags/postgresql));
* LDAP server (e.g. [OpenLDAP](https://www.digitalocean.com/community/articles/how-to-install-and-configure-a-basic-ldap-server-on-an-ubuntu-12-04-vps));
* SMTP server (e.g. [Postfix](https://www.digitalocean.com/community/articles/how-to-install-and-setup-postfix-on-ubuntu-12-04));
* IMAP server (e.g. Dovecot).

## Assumptions

This guide  assumes that:

* All of the components listed above are running on the same server (i.e. `localhost` or `127.0.0.1`); and
* That SOGo will be installed on the same server.
* You will be executing all of the commands that follow as the `root` user. You can switch from a local user to the `root` user by executing:

		sudo su

* You have previously installed the `vim` text editor:

		apt-get -y install vim

	Within `vim`, the keystrokes you'll use the most are:

	* `i` (insert)
	* `Esc` (Escape key)
	* `:` (colon)
	* `w` (write)
	* `q` (quit)
	* The arrow keys (to manipulate the cursor)
	* Finally, the `Backspace` & `Delete` keys 

## Preparatory Steps

First, check that your hostname and Fully Qualified Domain Name (FQDN) are properly set, by executing (which should yield a result such as `mail` (or any hostname of your choice):

	hostname

Then, execute:

	hostname -f

Which should yield a result such as `mail.yourdomain.tld`.

#### Edit Hostname

A server's hostname can be edited by executing:

	vim /etc/hostname

Then, tapping on the `i` key; followed by typing the hostname of your choice. To save & exit, tap the `Esc` key; then `:`; followed by `w`; `q` and `Enter`.

#### Edit FQDN

To edit a server's FQDN, execute:

	vim /etc/hosts

Then, tap on the `i` key and modify your hosts file so that it resembles the following (**obviously,** substituting the [hostname], [yourdomain], [tld], and [YourIP] values):

	127.0.0.1	localhost.localdomain	localhost
	127.0.1.1	hostname.yourdomain.tld	hostname
	YourIP		hostname.yourdomain.tld	hostname

#### Reload Hostname and/or FQDN

The following step is only necessary if you edited either the server's (i) hostname or (ii) FQDN:

	service hostname restart

### Add SOGo Repository & GPG Public Key

Append the SOGo repository to your `apt source list`, by copying & pasting both lines, below, into the command line and pressing `Enter`:

	echo -e "deb http://inverse.ca/ubuntu-nightly precise precise\n\
	deb-src http://inverse.ca/ubuntu-nightly precise precise" > /etc/apt/sources.list.d/SOGo.list

Next, you must add SOGo's GPG public key to Ubuntu's `apt keyring`. To do so, execute the following commands:

	apt-key adv --keyserver keys.gnupg.net --recv-key 0x810273C4

Then, update your lists of available software packages, by executing:

	apt-get update 

## Active Directory/ Domain Controller

Install the Samba4 packages and 

	apt-get -y install acl attr samba4 winbind4 krb5-user

Next, add the options `acl` and `user_xattr` to your mount point by executing:

	vim /etc/fstab

Then, tap on the `i` key and modify your settings so that it resembles the following:

	LABEL=DOROOT	/	ext4	acl,user_xattr,errors=remount-ro		0	1

Next, execute:

	mount / remount

Given that some useful files are missing from the Samba4 package found in the repository, we'll recompile it by executing the following commands, one-by-one:

	mkdir ~/Samba4
	cd ~/Samba4
	apt-get install dpkg-dev
	apt-get build-dep samba4
	apt-get source samba4
	cd samba4-4.0.1+dfsg1/
	dpkg-buildpackage
	make install

Next, delete the initial configuration generated by the Samba4 package:

	rm /etc/samba/smb.conf
	rm -R /var/lib/samba/private/*
	rm -R /var/lib/samba/sysvol/*

The `BIND` package was automatically installed when we installed Samba4. However, it can be uninstalled because Samba's internal DNS server will be used, instead:

	apt-get -y remove --purge bind9

Now, you are ready to provision Samba4 with your environment's variables (replace `YOURDOMAIN`, `yourdomain.tld`, and `SambaPWD` with values of your choice):

	 samba-tool domain provision --domain=YOURDOMAIN --adminpass=SambaPWD --dns-backend=SAMBA_INTERNAL --server-role=dc --function-level=2008_R2 --use-xattr=yes --use-rfc2307 --realm=yourdomain.tld

The script will generate your directory for Samba4. In the end, it will present a summary like this:

	Once the above files are installed, your Samba4 server will be ready to use
	Server Role:           active directory domain controller
	Hostname:              server
	NetBIOS Domain:        YOURDOMAIN
	DNS Domain:            yourdomain.tld
	DOMAIN SID:            S-1-5-21-1716394670-2609614925-1081274792

In order to have your domain and sharing, you have to manually change the configuration file with the command:

	vim /etc/samba/smb.conf

then add the following lines in the `[global]` section:

	template shell = /usr/sbin/nologin
	template homedir = /home/%ACCOUNTNAME%

Change your DNS in order to use Samba4 as main DNS server on your server, by executing:

	vim /etc/resolv.conf

Then, tap on the `i` key and modify your settings so that it resembles the following:

	domain yourdomain.tld
	search yourdomain.tld
	nameserver 127.0.0.1
	nameserver 8.8.8.8
	nameserver 8.8.4.4

Now, restart Samba4:

	service samba4 restart

## OpenChange Installation

To install OpenChange, execute:

	apt-get -y install openchangeserver sogo-openchange openchangeproxy openchange-ocsmanager openchange-rpcproxy

Next, provision Samba4 with the OpenChange schema:

	openchange_provision

Then, create an internal database for OpenChange:

	openchange_provision --openchangedb

Now, enable the MAPI procotol in Samba4:

	vim /etc/samba/smb.conf

Add the following lines somewhere in [global] section of the configuration file:

	dcerpc endpoint servers = +epmapper, +mapiproxy
	dcerpc_mapiproxy:server = true
	dcerpc_mapiproxy:interfaces = exchange_emsmdb, exchange_nsp, exchange_ds_rfr

## SOGo Installation

Sogo is the “backend“ of OpenChange. Install it with the following command:

	apt-get -y install sogo

## Install PostgreSQL Database

	apt-get -y install postgresql sope4.9-gdl1-postgresql

Next, create the SOGo database in PostgreSQL:

	su - postgres
	createuser --no-superuser --no-createdb --no-createrole --encrypted --pwprompt sogo
	createdb -O sogo sogo
	exit
	echo "host sogo sogo 127.0.0.1/32 md5" >> /etc/postgresql/9.1/main/pg_hba.conf

Finally, restart PostgreSQL:

	service postgresql restart

## Configure SOGo

	su - sogo -s /bin/bash
	defaults write sogod SOGoTimeZone "America/Chicago"
	defaults write sogod OCSFolderInfoURL "postgresql://sogo:PostgreSQL_pwd@localhost:5432/sogo/sogo_folder_info"
	defaults write sogod SOGoProfileURL "postgresql://sogo:PostgreSQL_pwd@localhost:5432/sogo/sogo_user_profile"
	defaults write sogod OCSSessionsFolderURL "postgresql://sogo:PostgreSQL_pwd@localhost:5432/sogo/sogo_sessions_folder"
	defaults write sogod OCSEMailAlarmsFolderURL "postgresql://sogo:PostgreSQL_pwd@localhost:5432/sogo/sogo_alarm_folder"
	defaults write sogod SOGoUserSources '({CNFieldName = displayName;  IDFieldName = cn; UIDFieldName = sAMAccountName; IMAPHostFieldName =; baseDN = "cn=Users,dc=yourdomain,dc=tld"; bindDN = "cn=Administrator,cn=Users,dc=youdomain,dc=tld"; bindPassword = SambaPWD; canAuthenticate = YES; displayName = "Shared Addresses"; hostname = "localhost"; id = public; isAddressBook = YES; port = 389;})'
	defaults write sogod WONoDetach YES
	defaults write sogod WOLogFile -
	defaults write sogod WOPidFile /tmp/sogo.pid
	defaults write sogod SOGoDraftsFolderName "Drafts"
	defaults write sogod SOGoSentFolderName "Sent"
	defaults write sogod SOGoTrashFolderName "Trash"
	defaults write sogod SOGoIMAPServer "localhost:144"
	defaults write sogod SOGoSieveServer "sieve://127.0.0.1:4190"
	defaults write sogod SOGoSieveScriptsEnabled "YES"

#### Optional

If you want to allow users to add their own IMAP account in SOGo, add the following command:

	defaults write sogod SOGoMailAuxiliaryUserAccountsEnabled YES

Logout of the `sogo` user & return to the `root` user

	exit

Create a symbolic link to allow Samba4 to use the SOGo configuration file:

	ln -s ~sogo/GNUstep ~root/GNUstep

There is a small bug in the `init.d` of Sogo that holds up the start-up process. You must edit the `init` file:

	vim /etc/init.d/sogo

and add the `-b` argument at lines 70 and 88:

	#Line 70
	if ! start-stop-daemon -b -c $USER --quiet --start --pidfile $PIDFILE --exec $DAEMON -- $DAEMON_OPTS
	# Line 88
	start-stop-daemon -b -c $USER --quiet --start --pidfile $PIDFILE --exec $DAEMON -- $DAEMON_OPTS

#### Restart SOGo & Samba4:

	service samba4 restart && nohup /etc/init.d/sogo restart &

## Cyrus IMAP Installation

The installation of Cyrus-Imap is done with the following command:

	apt-get -y install cyrus-admin-2.4 cyrus-imapd-2.4 sasl2-bin

#### Configure saslauth Authentication

Cyrus needs to use Saslauth system in order to authenticate its users. All small setup of Sasl in order to use Samba4.

	vim /etc/default/saslauthd

and change the following lines:

	...
	START=yes
	...
	MECHANISMS="ldap"

Create the following file with the command

	vim /etc/saslauthd.conf

and paste the following content and by changing of course the Administrator password:

	ldap_servers: ldapi://%2Fvar%2Flib%2Fsamba%2Fprivate%2Fldapi
	ldap_search_base: dc=yourdomain,dc=tld
	ldap_filter: (cn=%u)
	ldap_version: 3
	ldap_auth_method: bind
	ldap_bind_dn: Administrator@yourdomain.tld
	ldap_bind_pw: pass1234
	ldap_scope: sub

Restart the service:

	service saslauthd restart

You can also check that you authentication works:

	testsaslauthd -u administrator -p pass1234

#### Cyrus Configuration

	vim /etc/cyrus.conf

Add the following line in SERVICES section:

	imapnoauth      cmd="imapd -U 30 -N" listen="127.0.0.1:144" prefork=0 maxchild=100

## Additional Resources

* [Link1]();
* [Link2]();
* [Link3]().

# ~~~~~~~~~~ DRAFT ~~~~~~~~~~
*[Pull Requests](https://github.com/DigitalOcean-User-Projects/Articles-and-Tutorials/pulls) gladly accepted* 