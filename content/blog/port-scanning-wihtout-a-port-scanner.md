+++
title = "port scanning wihtout a port scanner"
date = "2011-11-25"
slug = "2011/11/25/port-scanning-wihtout-a-port-scanner"
Categories = []
+++

Booya.

For older bash versions

```bash 
for i in $(seq 1 1 1024); 
do 
echo > /dev/tcp/10.10.10.10/$i; 
[ $? == 0 ] && echo $i >>/tmp/open.txt; 
done
```

Same thing, newer bash versions

```bash 
for i in {1..1024}; 
do 
echo > /dev/tcp/10.10.10.10/$i; 
[ $? == 0 ] && echo $i >>/tmp/open.txt; 
done
```
