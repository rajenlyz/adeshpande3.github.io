---
layout: post
title: Rsync Usage for Backup
tags: [rsync, backup]
---

`rsync` is a tool for remote synchronization. The most simple usage is copy a remote/local file to locality/remote. This article, aims to complete a periodically backup mechanism(BUM) with rsync.

Firstly, introducing some info of rsync.

* installation
* [usage](https://download.samba.org/pub/rsync/rsync.html)

> 
Usage: rsync [OPTION]... SRC [SRC]... DEST
  or   rsync [OPTION]... SRC [SRC]... [USER@]HOST:DEST
  or   rsync [OPTION]... SRC [SRC]... [USER@]HOST::DEST
  or   rsync [OPTION]... SRC [SRC]... rsync://[USER@]HOST[:PORT]/DEST
  or   rsync [OPTION]... [USER@]HOST:SRC [DEST]
  or   rsync [OPTION]... [USER@]HOST::SRC [DEST]
  or   rsync [OPTION]... rsync://[USER@]HOST[:PORT]/SRC [DEST]
The ':' usages connect via remote shell, while '::' & 'rsync://' usages connect
to an rsync daemon, and require SRC or DEST to start with a module name.

* configuration: `/etc/rsyncd.conf`

Secondly, pre-conditions consist of:

* rsync installed
* rsync server ip: 10.155.12.6
* rsync server dir: /tmp/backup-rs
* rsync client ip: 10.155.12.8
* rsync client dir: /tmp/rajen-rs

## Rsync Server

In the rsync server, there are three steps to setup BUM server.

* config /etc/rsyncd.conf
* config /etc/rsyncd.secrets
* start rsync daemon

configuration file: */etc/rsyncd.conf*

```
## Specific the user to run the rsync daemon.
## The value need to be consistant with owner of dest dir(path=?), 
## or an error will be occured that: 'rsync:mkstemp failed:Permission denied'.
uid = rajen
gid = rajen

## disable or enable the 'chroot'
use chroot = yes
## specify the maximum number of simultaneous connections
max connections = 4
## specifies the file to use to support the "max connections"
lock file = /var/run/rsync.lock
pid file = /var/run/rsyncd.pid
log file = /var/log/rsyncd.log
## override the default port the daemon will listen on, defaults to 873
# port = 873

## module: backup
[backup]
## this dir must be exist, or 'chroot' will be error.
path = /tmp/backup-rs/
ignore errors
## enable client to upload file
read only = false
list = false
## which address allowed to connect, can be combine with 'hosts deny'.
hosts allow = 10.155.12.8/32
## the user name can be self-defined.
auth users = rjtest
secrets file = /etc/rsyncd.secrets
```

More detailed explaination is [here](https://download.samba.org/pub/rsync/rsyncd.conf.html)

configration file: */etc/rsyncd.secrets*

the format of secret file is like this: `${auth-user}:${passwd}`.
i.e. 
```
$ cat /etc/rsyncd.secrets
rjtest:abcdw
```
Take care, the rsyncd.pwd mode need to be modified to `600`,
because password file must not be other-accessible.
```
$ sudo chmod 600 /etc/rsyncd.secrets
```

rsync daemon can be started like this: `sudo rsync --daemon`.

But by this method, when the server is reboot, the process is unable to self-started. So to give the self-started role, follow the below step:
```
$ echo >> "sudo rsync --daemon" >> /etc/rc.d/rc.local
```

## Rsync Client

Before upload files, firstly add a configuration for rsync password.
```
$ cat /home/rajen/rsyncd.secrets
abcdw
```

In the rsync client, using a single command to upload files to server.
```
rsync -vrtL --progress /tmp/rajen-rs/* rjtest@10.155.12.6::backup \
--password-file=/home/rajen/rsyncd.secrets

```

when executing this command, rsync will connect to the rsync daemon in remote server(10.155.12.6), with the user:rjtest and password:abcdw.

For periodical task, it need to create a crontab task.

create a shell script：
```
$ cat /home/rajen/script/rsync.sh
#!/bin/bash
rsync -vrtL --progress /tmp/rajen-rs/* rjtest@10.155.12.6::backup \
--password-file=/home/rajen/rsyncd.secrets
$ chmod 600 /home/rajen/script/rsync.sh
```

create a crontab task:
```
$ crontab -e
*/30 * * * * /home/rajen/script/rsync.sh
```
