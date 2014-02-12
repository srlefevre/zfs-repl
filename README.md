zfs-repl
========
A multi-platform ZFS snapshot and replication tool supporting compression and encryption over multiple transport protocols.

Purpose
=======
This tool/script will create and replicate ZFS datasets/file systems using snapshots between source and target hosts on *nix systems.  Its intended to be run primarily via cron as the root user but can be run by other users (with appropriate privileges) from the command line.

The tool has been written to be flexible so that it can run on and against hopefully any *nix system that supports ZFS storage pools.

This system has been tested on the following platforms:
* CentOS 6.5
* Ubuntu 12.04


Background/History
==================
This script is a fork/modification of others prior work.  Specifically, the work of 
- Mike La Spina
- kattunga

I've simply polished the script and added a few features.


Prerequisites
===========
1. You know what your doing 
   * You know how to setup ssh to use public/private key authentication 
   * You know how to setup a user under sudo to not prompt for a password
2. Both source and target systems are running ZFS pools
3. User on source system is root or can run sudo.
4. User on target is root or can run sudo without a password for the zfs command.
5. Source system has a functioning ssh client.
6. Target systems have a functioning ssh server.
7. The user on source system can ssh to target system using public/private keys without prompting for a password (e.g. 'ssh repl-user@target')
8. Source system has GNU date command installed.  (This is needed to handle aging/deletion of snapshots)

(Optionally) The following commands are installed on both source and target systems:

|Feature/Capability | Source system | Target system |
|-------------------|---------------|---------------|
|Compression | gzip | zcat |
| | xz | xzcat |
|Transport Protcols | nc | nc |
| | socat | socat |
| | mbuffer | mbuffer |
|Encryption | openssl | openssl |
|Transfer Stats | pv | - |
|Automatic Port Selection | shuf | - |
|Log to syslog | logger | - |
|Email logs/errors | mailx | - |

Use the package manager for your system to install the required and optional commands (e.g. yum, apt-get, pkg)


Setup
=====
For those that pace in front of microwaves...

Install
-------
```
git clone https://github.com/srlefevre/zfs-repl.git
cd zfs-repl
sudo cp zfs-repl /usr/local/bin/
sudo mkdir /etc/zfs-repl
```

Configure
---------

Create the configuration files for the source and target systems and then edit the configuration files using your preferred text editor.

```
sudo ./source-config > /etc/zfs-repl/zfs-repl.conf
sudo ./target-config --host [[user@]target-name] > /etc/zfs-repl/[target-name].conf
sudo nano /etc/zfs-repl/zfs-repl.conf  
```

**zfs-repl.conf**

The /etc/zfs-repl/zfs-repl.conf file is a simple way to configure the script without having to edit the code directly.  The main thing that is needed is to specify the full path to each command.  The 'source-config' and 'target-config' scripts will handle most of this for you.  Please check to ensure DATE points to the GNU date command in the zfs-repl.conf file.

Minimally, in /etc/zfs-repl/zfs-repl.conf you need to specify 
* the email addresses you want to use to email errors/logs from and to 
* the lock path and log file

```
# email settings
mail_from=root@example.com
mail_to=user@example.com
mail_subject=zfs-replication

#process files/paths
LOCK_PATH=/var/lock
LOG_FILE=/var/log/zfs-repl.log
```
You may need to change the LOCK_PATH and LOG_FILE location to match the standard locations for your *nix installation.
LOCK_PATH could be set to /var/lock, /var/lock/subsys, or even /tmp

You can also specify global default settings in the /etc/zfs-repl/zfs-repl.conf. 



**[target-name].conf**

*Note:* For the uninitiated, [target-host] should be replaced by the hostname of your target system. For example, if you specify --host repl@nas12 on the 'zfs-repl' command line then nas12 would be your hostname and the configuration file would be /etc/zfs-repl/nas12.conf

The /etc/zfs-repl/[target-name].conf file holds configuration setting specific to the target host. The basic setup should be completed using the 'target-config' command as stated earlier.  It can also be used to set defaults for the specific host.  For example, if you always want to use mbuffer, gzip, and encryption when replicating to a specific host then add the following to [target-name].conf file.

```
PROTOCOL=MBUFFER
COMPRESSION=GZIP
ENCRYPT=true
```

If you need to replicate to more then one host, make sure you run the 'target-config' command for each host and create the appropriate .conf file for each host.  For example:

```
sudo ./target-config --host repl@host1 > /etc/zfs-repl/host1.conf
sudo ./target-config --host root@host2 > /etc/zfs-repl/host2.conf
```


Usage
=====

See 'zfs-repl --help'.

*Example 1:*
Source system hostX needs to backup the every changing file system pool0/data01 to the target system hostZ under tank1/backup/data01.  hostX wants to maintain a rolling two days of snapshots but hostZ needs to maintain seven days of snapshots.  Snapshots need to be taken and replicated every hour using netcat transport protocol with gzip compression.  Further, hostZ needs to compress the backup file system using lz4.

On hostZ, at least zpool tank1 should already be created but tank1/backup/data01 should not.

Initial replication from hostX as root
```
# zfs-repl --host repl@hostZ --source pool0/data01 --dest tank1/backup/data01 --create-dest --dest-opt compression=lz4 --protocol NETCAT --compression GZIP 
```
N.B. If pv (pipeview) is installed, you can add the '--stats' option to see how the replication progresses.


Hourly snapshot replication crontab entry
```
0 * * * *  /usr/local/bin/zfs-repl --host repl@hostZ --source pool0/data01 --dest tank1/backup/data01  --protocol NETCAT --compression GZIP --snap-retain "-2 days" --dest-snap-retain "-7 days"
```


More examples to be added later.


Notes
=====

mbuffer
-------

mbuffer is available on many *nix distrobutions as part of the base operating system.  On Red Hat EL6/CentOS 6 and other derived linux distrobutions, it is not included.  I was able to find mbuffer from Federa Core 13 works on CentOS 6 without issue.


