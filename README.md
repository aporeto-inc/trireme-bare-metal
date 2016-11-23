# trireme-cumulus

We describe some basic configuration for enabling host routing to advertise docker networks.

# Host Configuration

## Configure Docker 

This is an *one-time* configuration that is needed in every host. Essentially we need to configure
a routable subnet where all the containers will be attached to. Note that there is only one restriction
on what is the subnet IP. Since we will use routing to advertise the subnet it has to be 
unique in your data center network. Note also that the advertisement will only happen when a host 
is activated and not every time a new container is added or removed in the data center. 

### Using the daemon configuration 

The host must be configured with the bridge that all Docker containers will be activated on. This can be done
either by creating a new network using Docker or by simply starting the docker daemon with the 
correct parameters. We always recommend to disable the userland proxy of Docker since its extremely
slow. We recommend the following mechanism to start the Docker daemon. This can be achieved
by configuring your systemctl controls. The easiest way to find the file is to do :

```bash 
sudo systemctl status service 
● docker.service - Docker Application Container Engine
   Loaded: loaded (/lib/systemd/system/docker.service; enabled; vendor preset: enabled)
   Active: active (running) since Tue 2016-11-22 16:31:07 PST; 59s ago
     Docs: https://docs.docker.com
 Main PID: 6560 (dockerd)
   Memory: 17.8M
      CPU: 329ms
   CGroup: /system.slice/docker.service
           ├─6560 /usr/bin/dockerd -H fd:// --userland-proxy=false --bip=172.0.0.1/24
           └─6567 docker-containerd -l unix:///var/run/docker/libcontainerd/docker-containerd.sock --shim docker-containerd-shim --metrics-interval...
```
In the above example the configuration file is in /lib/systemd/system/docker.service

You can edit the file and replace the ExecStart line with the following

```bash
/usr/bin/dockerd -H fd:// --userland-proxy=false --bip=10.1.1.1/24
```

Restart your docker daemon 

```bash 
sudo systemctl daemon-reload
sudo systemctl restart docker
```

If you look at your bridge configuration now for docker0 you will see that its IP address matches
the address above 

```bash
user@ubuntu:~$ ip add show docker0
5: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:61:dc:8b:ab brd ff:ff:ff:ff:ff:ff
    inet 10.1.1.1/24 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:61ff:fedc:8bab/64 scope link
       valid_lft forever preferred_lft forever
```

### Using Docker libnetwork 

Alternatively, you can create a new network using the docker libnetwork capability. This will essentially
add a new bridge in the host and give it the specified IP address

```bash
docker network create  \
           --gateway 10.1.1.1 \
           --subnet 10.1.1.0/24 \
           --driver bridge \
           -o "com.docker.network.bridge.enable_ip_masquerade=false”  containers
```

The above configuration will survive reboots. However, if you use this approach you have to make sure 
that you start your containers with the "--net" parameter. As an example

```bash
docker run --net=containers -d nginx 
```

## Configure Host Routing 
We will use the techniques described in https://cumulusnetworks.com/routing-on-the-host/ to create a simple
example. More complex routing configurations can be obviously implemented.

For simplicity in this example we will use the Cumulus docker container to configure a local routing protocol
on the host that will advertise the container network to the rest of the world. This configuration can 
be automated and streamlined and is only needed once a host is booted. In production environments one 
can choose to use the Cumulus Quagga implementation without a docker container. 

First start the Cumulus Router 

```bash 
docker run -t -d --net=host --privileged --name Quagga cumulusnetworks/quagga:denial-latest
```

Note, that the router needs to start in privileged mode in order to be able to modify the routing tables. 
In this example, we will assume that the network interface of the host has some IP address 10.2.1.2 
and that the corresponding router interface is 10.2.1.1. One can use techniques with unumbered interfaces
and DHCP to distribute this IP to the hosts. 

We first need to start the Quagga services
```bash
docker exec Quagga /usr/lib/quagga/quagga restart
```

We then need to configure BGP to communicate with the ToR switch
```bash 
docker exec Quagga /usr/bin/vtysh  \
       -c 'configure t' \
       -c 'router bgp 1000' \
       -c 'neighbor 10.2.1.1 remote-as 10000' 
       -c 'address-family ipv4 unicast' 
       -c 'network 10.1.1.0/24' 
       -c 'neighbor 10.2.1.1 activate’
```
In the above configureation we did the following:
* Started BGP on the AS 1000 for IPv4
* Connected to the neighbor with IP 10.2.1.1
* Advertised to the neighbor the container route 10.1.1.0/24
* Activated the communication

At this point the configuration of the host is complete.

# Configure the Leaf Switch 

Configuring the Leaf Switch is straightforward. We assume that the interface IP is 10.2.1.1. Note that we could 
use a loopback IP to connect the BGP processes as well. 

The configuration is 
```bash
configure t
router bgp 10000
neighbor 10.2.1.2 remote-as 1000
address-family ipv4 unicast
neighbor 10.2.1.2 activate
exit 
exit
```
Essentially its a simple symetrical communication. 

# Operation
If you look at the routes advertised to the Leaf Switch you will notice that the container network is not there. Indeed,
the host router will only advertise this network if at least one container is attached to the network. In order to 
verify, simply create a container 

```bash
docker run --net=containers -d --name web nginx 
docker inspect web 
```

From the docker inspect command get the IP address of the container and go to your 
Leaf switch and you can now ping the container. You should be able to simply access the 
nginx web server from anywhere in your data center.
