---
layout: post
title: Retrieving Safari History From JailBroken IOS Device 
date: 2015-06-28 07:10:58 UTC
updated: 2015-06-28 07:10:58 UTC
comments: true
categories: ios jailbreak safari history homepage
---

### This post is for Jail Broken devices and tested on IOS 8.1.2. ###

This post explains how to retrieve safari history database from a jail broken ios device. Its a sqlite3 database file which contains all history data but the challenge is that there is no specific path on which the files exist, so you will have to first find the files.

1 Install OpenSSH on your jailbroken device (through Cydia).

2 Find History.db* files

Application data for most applications reside in subdirectories under:

```
/private/var/mobile/Containers/Data/Application/
```
The files which contain safari history data are:

```
History.db
History.db-shm
History.db-wal
```
To find path to these files just do:

```
$ find /private/var/mobile/Containers/Data/Application -iname 'History.db*'
```

3 SCP files to a local location

Basically to get the data from these files you need `sqlite3`. You can either install `sqlite3` app from cydia to the mobile or better just scp the datbase files to machine you are working on.

You can copy the files one by one or better use `sshpass` and `xargs` to copy all files at one go like:

```
$ sshpass -p 'alpine' ssh  root@mobile_ip "find /private/var/mobile/Containers/Data/Application -name 'History.db*'" \
    | xargs -I{} \
    sshpass -p 'alpine' scp -P $SSH_PORT root@mobile_ip:'{}' /path/to/tmp/db/on/local/machine;
```

4 Load the sqlite database

If you change your working directory to the directory containing `History.db*` files and do:

```
$ sqlite3 History.db
```

It will load sqlite3 database and now you can get any data you want about browser history.

5 Example query to get latest page visited:

```
> select url from history_items where id in( select history_item from history_visits order by visit_time desc limit 1);
```

That's it ! Now you can just play around and get stats like number of times and time when a website was visited and much more.
