# 42_BADASS
p1

run frrouting image 
```
docker run -it --name frr-test frrouting/frr:latest /bin/sh
```
modify config for 
```
vi /etc/frr/daemons
```
bgpd=yes
ospfd=yes
isisd=yes

in my host
docker cp frr-test:/etc/frr/daemons ./

