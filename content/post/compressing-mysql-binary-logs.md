---
title: "compressing mysql binary logs"
date: "2013-01-14"
slug: "2013/01/14/compressing-mysql-binary-logs"
categories:
  - devops
  - sql
  - linux
cover: "/images/mysql.png"
---

Under normal circumstances, master servers in a replication can be setup to automatically rotate binary logs using the expire_logs_days my.cnf configuration setting.

However when it is known that slaves are in sync, it can be beneficial to pro-actively reduce on-disk size using compression. This can be especially useful in high-churn environments where binary logs grow quickly.

Grab the script:

```bash
git clone git://github.com/marksteele/mysql-dba-tools.git
```
<!--more--> 
Of course you need to be aware that to create a new slave (eg: using innobackup) you'll need to stop this script or copy the compressed binary logs to the slave and feed them in.

Also note, the --purge option should pretty much always be set (otherwise there's not any point in running this script).


```bash
root@dbserver1# ./rotatelogs.pl --purge \
--numslaves=2 --host=localhost --user=root --pass=mypass \
--datadir=/var/lib/mysql --priority=10 --keep=4
```

It works by connecting to the DB, grabbing the current binary log, checking that the slaves are all connected and currently synched. Once it.s confirmed that the slaves are ok,it starts looping over the logs, compressing and purging them. Once the current set is processed, it checks the list of compressed logs and rotates out old logs
