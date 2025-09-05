## P1

In P1, we introduce two Docker images:

* A lightweight FRRouting image, with the necessary routing daemons.
* A simple host based on an Alpine image. 

The goal of this exercise is to make them communicate with each other.

### Steps

In your host terminal:
```
# create the Docker images
docker build -t router_wlo -f router_wlo .
docker build -t host_wlo -f host_wlo .
```
In the auxiliary console of the router image:
```
# set the IP address:
ip addr add 192.168.1.1/24 dev eth0

# turn on the device
ip link set eth0 up
```

In the auxiliary console of the host image:
```
# set the IP address:
ip addr add 192.168.1.2/24 dev eth0

# test if the images are connected:
ping 192.168.1.1
```