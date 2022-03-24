# Docker-Networking-Bridge-Network

In this story, I will talk about docker networking more specifically about bridge networking in which is the common and default built in driver in docker.
The built-in drivers in docker installation include:

- bridge (default)

- null — For standalone containers, remove network isolation between the container and the Docker host, and use the host’s networking directly.

- host — For this container, disable all networking. Usually used in conjunction with a custom network driver.

```
$ sudo docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
6965b9d0e60d        bridge              bridge              local
def783a1384d        host                host                local
2817e7b82eec        none                null                local
```

You can read more about the drivers here — https://docs.docker.com/network/

## Bridge Network

In bridge network, the docker creates a private internal network. All the containers will get an IP address and they can communicate with each other.

You can see the output of ifconfig just before and after installation of docker daemon.

```
# ifconfig
ens4: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1460
        inet 10.211.40.4  netmask 255.255.255.255  broadcast 10.211.40.4
        inet6 fe80::4001:aff:fed3:2804  prefixlen 64  scopeid 0x20<link>
        ether 42:01:0a:d3:28:04  txqueuelen 1000  (Ethernet)
        RX packets 1255511  bytes 320406571 (305.5 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1376736  bytes 118900731 (113.3 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 36  bytes 5796 (5.6 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 36  bytes 5796 (5.6 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

```
# ifconfig
docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        ether 02:42:45:bd:34:e0  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
ens4: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1460
        inet 10.211.40.4  netmask 255.255.255.255  broadcast 10.211.40.4
        inet6 fe80::4001:aff:fed3:2804  prefixlen 64  scopeid 0x20<link>
        ether 42:01:0a:d3:28:04  txqueuelen 1000  (Ethernet)
        RX packets 1539910  bytes 458721956 (437.4 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1686829  bytes 145677782 (138.9 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 36  bytes 5796 (5.6 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 36  bytes 5796 (5.6 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

Here you can see docker0 ethernet came into picture after installation of docker. It means, docker has created a virtual ethernet.

So whenever docker image is pulled it uses the default bridge network and is created within `docker0` network.

```
$ docker ps
CONTAINER ID   IMAGE     COMMAND                  CREATED       STATUS       PORTS                                   NAMES
6e5b3e15719f   nginx     "/docker-entrypoint.…"   6 hours ago   Up 6 hours   0.0.0.0:8000->80/tcp, :::8000->80/tcp   hungry_snyder

~$ docker inspect --format '{{ .NetworkSettings.IPAddress }}' 6e5b3e15719f
172.17.0.2
```

Each container will get its own IP address and can talk each other.


## Communication with External World

Docker relies on iptables to configure its networking. This includes NAT rules to handle access to and from the external network, and lots of other rules to configure containers access to each other on docker networks. By default, this access is open, but there are options when creating networks to restrict outside access and inter container communication.

Below are the rules written in iptables by docker.

```
# iptables -nL
Chain INPUT (policy ACCEPT)
target     prot opt source               destination
Chain FORWARD (policy DROP)
target     prot opt source               destination
DOCKER-USER  all  --  0.0.0.0/0            0.0.0.0/0
DOCKER-ISOLATION-STAGE-1  all  --  0.0.0.0/0            0.0.0.0/0
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0
Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
Chain l (0 references)
target     prot opt source               destination
Chain DOCKER (1 references)
target     prot opt source               destination
Chain DOCKER-ISOLATION-STAGE-1 (1 references)
target     prot opt source               destination
DOCKER-ISOLATION-STAGE-2  all  --  0.0.0.0/0            0.0.0.0/0
RETURN     all  --  0.0.0.0/0            0.0.0.0/0
Chain DOCKER-ISOLATION-STAGE-2 (1 references)
target     prot opt source               destination
DROP       all  --  0.0.0.0/0            0.0.0.0/0
RETURN     all  --  0.0.0.0/0            0.0.0.0/0
Chain DOCKER-USER (1 references)
target     prot opt source               destination
RETURN     all  --  0.0.0.0/0            0.0.0.0/0
```

Publishing a port on the host implicitly allows access from outside.

## Conclusion

Hope you understood the basic of how docker communicate with each other and to/from the outside world.
Happy day!

