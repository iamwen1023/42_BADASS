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

docker build -t router_wlo -f router_wlo ." -t and -f 

Configure IP addresses

In MinimalNode:
ip addr add 192.168.1.2/24 dev eth0
ip link set eth0 up

In RouterNode:
ip addr add 192.168.1.1/24 dev eth0
ip link set eth0 up


Right-click the node → Configure → Advanced → Startup script

Add your commands:
ip addr add 192.168.1.1/24 dev eth0
ip link set eth0 up


Test and check ps
Ping from MinimalNode:
ping 192.168.1.1

