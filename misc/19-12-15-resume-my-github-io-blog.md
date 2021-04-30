---
layout: post
title:  "Resume personal site with Jekyll on Docker"
subtitle: "Network troubleshooting for docker-for-mac client"
date:  2019-12-29 12:27:42
categories: [docker, network, jekyll]
tag: [docker, jekyll, network]
---

TL;DR, I should've used this pre-configured [jekyll image](https://github.com/BretFisher/jekyll-serve) at the very beginning to avoid all the troubles I met. And this image is quite easy to use:

```bash
cd dir/of/your/jekyll/site
# it will mount your current path into the containers /site,
# bundle install before running jekyll serve to
# serve it at http://<docker-host>:8080.
docker run -p 8080:4000 -v $(pwd):/site bretfisher/jekyll-serve
```

Bare metal installation of jekyll on a fresh Linux container is also viable. Just make sure the jekyll server is listening on [0.0.0.0, instead of 127.0.0.1][3].

If for security reasons you cannot use `0.0.0.0`, you can use [ssh tunneling to forward traffic to docker](https://stackoverflow.com/questions/52321269/how-to-reach-docker-container-localhost-from-mac).

---

## motivation to resume my blog

I used to build jekyll from scratch to build [guihao-liang.io][0]. I found the [personal static website template][1] is fairly easy to use (but I end up using an [another template](https://github.com/daattali/beautiful-jekyll#readme) just to post blogs). I decided to give it a try. First, I follow the [instructions][1] to fork a site for myself.

```bash
gem install jekyll bundler
cd <your-personal-website>
bundle install
bundle exec jekyll serve
```

Later on, I realize that this site is based on Ruby, which I don't want to install on my local machine because I'm not a Ruby developer nor a full stack.

There's one simplest way to avoid this hurdle is to just push local changes to my [github.io repo][2] and test site deployed by github manually. If the site is not expected, modify locally, push it to the repository again and wait until it's deployed (sometimes, 20 mins). Repeat this process until I'm satisfied with it. That's too troublesome for development. It's like there's no local test, and all the dev work is verified from production directly. If the user is upset, the further bug fix is on fire. That reminds me of some engineers that I've seen in my early career. Definitely, I don't want to be one of them.

Therefore, I realize I can have a docker container running to test locally because you can mess around with it and dispose it at your will without polluting the host machine.

---

## Install jekyll on docker from scratch

At the very beginning, I just pull a [Ubuntu docker image](https://hub.docker.com/_/ubuntu/) and run it as a container with an interactive shell. At the same time, I realize I have to forward network ([binding ports](https://runnable.com/docker/binding-docker-ports)) from my Mac host to the container.

```bash
# -p host_port:container_port (mnemonic: forward from host to container)
docker run -it --name blog -p 5000:80 ubuntu:latest /bin/bash
```

In the pseudo-tty, I manually `apt-get installed` a bunch of things I need, such as git, sudo, [zsh](http://www.boekhoff.info/how-to-install-zsh-and-oh-my-zsh/), [oh-my-zsh](https://ohmyz.sh/), and [Jekyll](https://jekyllrb.com/docs/installation/ubuntu/). Normally, you don't need those tools to be installed since the container is running without interactive shells. Here, I plan to use this environment as a long-term Linux box.

After installation is complete, I realize I don't have access to my [blog repo](https://github.com/guihao-liang/guihao-liang.github.io) from container instance to the host machine. The reason I need to have access to my host machine's file system is that I mainly edit my blogs on my Mac, where I have a nice [GUI](https://en.wikipedia.org/wiki/Graphical_user_interface). As far as I googled, there's no way to mount the host file system at runtime. The worst-case scenario is that I have to shut down the container and redo all the installations. At the time I planned to give up, I realize I can [commit](https://docs.docker.com/engine/reference/commandline/commit/) this container to save its state as an image for later use!

```bash
# docker commit -a author -m "message"  <container_name> tag
docker commit -a guihaol -m "my blog image" blog gblog:test
```

Neat, I don't have to start over. This time, I need to pass `-v` to tell docker which path I want to mount to the container to [share data with host](https://www.digitalocean.com/community/tutorials/how-to-share-data-between-the-docker-container-and-the-host).

```bash
# -v <path_of_host>:<mount_path_of_container>
docker run -it --name blog -p 5000:80 -v /path/to/blog:/blog gblog:test /bin/zsh
```

The command mounts local blog repo on my Mac host to the container's `/blog`. What I have to do next is to `cd` to `/blog` in the container, and follow [instructions][2] to start Jekyll:

```bash
gem install jekyll bundler
bundle install
bundle exec jekyll serve -p 5000
...

Server address: http://127.0.0.1:4000
Server running... press ctrl-c to stop.

```

I feel very excited to see my blog running locally! I provided `localhost:5000` to my browser, hitting enter key with extreme excitement, and I got `This site can’t be reached`. OMG! That means my network binding isn't right! How dare you!

### container network binding

I already bind the ports from host to container, but why the container cannot be reached?

Since I used the docker-for-mac client, I checked [networking features][4] documentation. It shows that I'm using the right command `-p` to do the port-binding to the host, which creates a [firewall rule](#maskquerade-rules) which maps a container port to a port on the Docker host.

What this [hello world](https://github.com/karthequian/docker-helloworld/blob/a28dd5245c9347905330d0b16c36ddee41af76e1/Dockerfile#L45) does is just use docker command `EXPOSE 80`, which should be the same thing as `-p`.

Under the hood, it forwards the port from host to container. And we can [get the port mappings](https://stackoverflow.com/questions/32444612/how-to-get-the-mapped-port-on-host-from-a-docker-container),

```bash
# blog is the container name
$ docker port blog
# docker -> host
8000/tcp -> 0.0.0.0:7070
```

### ping container using its IP

Okay, it's mysterious, probably I should find the IP address to ping that `ip:port` directly. Hold, on, the doc also says:

> Q: I cannot ping my containers<br>
> A: Docker Desktop for Mac can’t route traffic to containers.<br>
> Q: Per-container IP addressing is not possible<br>
> A: The docker (Linux) bridge network is not reachable from the macOS host.

### network drivers

```bash
$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
19f6b73480fb        bridge              bridge              local
f5a055f5b740        host                host                local
102cb83107c9        none                null                local
```

The `none` network driver is the simplest, which disable all networking of the container. Let's talk about `host` and `bridge`.

### host network

I was told if I use docker command option `--net=host`, the container can share the network with my host, which means the container IP address should be an address from the host and reachable. Referenced from the [doc](https://docs.docker.com/network/host/):

> If you use the host network mode for a container, that container’s network stack is not isolated from the Docker host (the container shares the host’s networking namespace), and the container does not get its own IP-address allocated. For instance, if you run a container that binds to port 80 and you use host networking, the container’s application is available on port 80 on the host’s IP address.

That should work but it didn't. When I continued to read the doc, it says **host networking driver only works on Linux host**. Sign.

### bridge network

<img src="https://docs.docker.com/docker-for-mac/images/docker-for-mac-install.png" width="80%">

[image source](https://docs.docker.com/docker-for-mac/images/docker-for-mac-install.png)

Bridge network is [used by default by docker](https://docs.docker.com/network/), but what is  `bridge network`? And [how container connects to host through networking](https://www.docker.com/blog/understanding-docker-networking-drivers-use-cases/)?

<img src="https://docs.docker.com/engine/tutorials/bridge1.png">

[image source](https://docs.docker.com/engine/tutorials/bridge1.png)

The initial guess of mine is that the [bridge network](https://en.wikipedia.org/wiki/Bridging_(networking)) device is on the link layer and aims to connect a group of hosts into a single local-area network (LAN). Based on my learning of computer networks, each IP node in the LAN maintains a [ARP](https://en.wikipedia.org/wiki/Address_Resolution_Protocol) table, which is the IP/MAC mappings. Within the LAN, network packets (or frames) are routed according to the host's MAC address. The `bridge` sounds like the switch device, which works at the link layer and only needs to know MAC/interface mappings to direct the packets to the right places.

> Containers connected to the default bridge network can communicate with each other by IP address. Docker does not support automatic service discovery on the default bridge network. If you want containers to be able to resolve IP addresses by container name, you should use user-defined networks instead.

Well, we cannot use `hostname` is probably due to the fact that no `dns` is bundled with this `bridge` network. We can only rely on the host's name resolution table,

```bash
# running as root by default in container
root@6dbee430cbd5:/guihao/Playground/jarvi-io# cat /etc/hosts
127.0.0.1       localhost
::1     localhost ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
172.17.0.2      6dbee430cbd5
```

Good, it has `172.17.0.2`, the container is running. Let's verify my initial guess by using `arp` command,

```bash
root@6dbee430cbd5# apt-get update && apt-get install net-tools

# 172.17.0.1 is the ip for the gateway device
root@6dbee430cbd5# arp -a
? (172.17.0.1) at 02:42:6b:93:ed:1b [ether] on eth0
```

Not very responsive to my guess since it doesn't have the running container's IP. Let's check the bridge network settings from docker,

```bash
$ docker network inspect bridge
[
    {
        "Name": "bridge",
        "Id": "19f6b73480fb56262850a5f53cba4f99c3a08a6c8c001f07cdae708f4cd5aab7",
        ...
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        ...
        "Containers": {
            "6dbee430cbd5ac28d6b23cd1099a519c52a70dd5fa20b4a7446d67737603768e": {
                "Name": "blog",
                "EndpointID": "186bf3fee3912f07a6d0b789e06cfbd8792c7d957339294bff91ff66796be5e9",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            }
        },
        ...
    }
]
```

Surprisingly, `172.17.0.1` is the IP for the `bridge` device. The `bridge` network serves like a gateway **router**, which works at layer 4, instead of the layer 3 **switch**. The output of `arp -a` in the container doesn't have other hosts' IP/MAC mappings but only the gateway's. I reckon the topology of this bridge network is star-shaped, where all hosts connect to the gateway router and this gateway router routes the traffic in and out. To conclude, there's no LAN.

Great! If we can talk to this gateway, we might be able to ping the container with its IP!

Continue from this [doc](https://docs.docker.com/docker-for-mac/networking/):

> Q: There is no docker0 bridge on macOS<br>
> A: Because of the way networking is implemented in Docker Desktop for Mac, you cannot see a docker0 interface on the host. This interface is actually within the virtual machine.

`docker0` is the network bridge driver that is used by the container, which we've seen above, and it's not visible by the host.

[xhyve vm inside docker-for-mac has no network adapter](https://stackoverflow.com/questions/41819391/can-not-ping-docker-in-macos) and the `bridge` device runs inside of VM on Mac. That's a bummer.

The question is that, if we cannot ping the IP, how could docker accept traffic? Well, similar to ssh tunneling, the docker client listens on all network interfaces and forwards the traffic to the VM locally, and the VM passes the traffic to the `bridge` network.

We can check the mapping by,

```bash
docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                            NAMES
6dbee430cbd5        gblog:test          "/bin/bash"         3 hours ago         Up 3 hours          80/tcp, 0.0.0.0:7070->8000/tcp   blog
```

and also check the ports on the host being listened,

```bash
# port `7070` is used for forwarding in this post
sudo lsof -iTCP -sTCP:LISTEN -P -n | grep 7070
COMMAND     PID        USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
com.docke  1365 guihaoliang   22u  IPv6 0xed002061bd873c21      0t0  TCP *:7070 (LISTEN)
```

Trace the route of an outgoing packet,

```bash
# apt-get install traceroute
root@88677a7f49f4:/# traceroute -n4 google.com
traceroute to google.com (172.217.3.174), 30 hops max, 60 byte packets
 1  172.17.0.1  1.049 ms  0.977 ms  0.938 ms
 2  192.168.0.1  2.506 ms  2.599 ms  3.240 ms
 ...
 11  209.85.254.170  16.770 ms 209.85.254.236  20.126 ms 74.125.253.60  20.166 ms
 12  74.125.243.194  20.091 ms 172.217.3.174  16.296 ms  15.709 ms
```

As a result, the packet first is sent to the gateway, the bridge network device, and then to the default wireless router's address `192.168.0.1`.

> 192.168.0.1 is a common Internet Protocol (IP) address for many wireless home routers, used to access administrative functions.

With the further reading of [docker port binding][5], the `bridge` works more like a [NAT](https://en.wikipedia.org/wiki/Network_address_translation) server, which governs inbound and outbound traffic for hosts hidden bebind of it. That's why container-wise IP is not reachable from outside but the container can talk with outside world.

> By default Docker containers can make connections to the outside world, but the outside world cannot connect to containers. Each outgoing connection will appear to originate from one of the host machine’s own IP addresses thanks to an **iptables masquerading** rule on the host machine that the Docker server creates when it starts

```bash
root@88677a7f49f4:/# iptables -t nat -L -n
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination

Chain INPUT (policy ACCEPT)
target     prot opt source               destination

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination

Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
```

In my container, I don't have any special rules for it and it can ping outside, like `google.com`, freely. To reproduce above output, you need to start container with extra [--cap-add=NET_ADMIN --cap-add=NET_RAW](https://stackoverflow.com/questions/41706983/installing-iptables-in-docker-container-based-on-alpinelinux)

#### iptables

What is the iptables?

From a [high level perspective](https://www.booleanworld.com/depth-guide-iptables-linux-firewall/),

It can add rules to filter or alter packets of inbound and outbound packets at different stages. The packet filtering mechanism provided by iptables is organized into three different kinds of structures: tables, chains and targets.

Tables contain rules to be executed in different chains (various points where packets are processed). Targets are corresponding reaction to process the matched packets.

<img src="https://www.booleanworld.com/wp-content/uploads/2017/06/Untitled-Diagram.png" width="80%">

[image source](https://www.booleanworld.com/wp-content/uploads/2017/06/Untitled-Diagram.png)

From [low level implementation perspective](https://github.com/moby/moby/issues/33605#issuecomment-307361421),

> iptables is a bit special because manipulating the rules and tables requires userland binaries (and libraries), but the packet processing is done entirely in the kernel. In most modern distros, the kernel side is compiled as modules. This means that as long as you don't use iptables, the kernel modules are not loaded.

#### maskquerade rules

Linux maskquerade is like NAT. A brief [intor to masquerade](https://www.tldp.org/HOWTO/IP-Masquerade-HOWTO/ipmasq-background2.1.html)

> IP Masquerade is a networking function in Linux similar to the one-to-many (1:Many) NAT (Network Address Translation) servers found in many commercial firewalls and network routers. For example, if a Linux host is connected to the Internet via PPP, Ethernet, etc., the IP Masquerade feature allows other "internal" computers connected to this Linux box (via PPP, Ethernet, etc.) to also reach the Internet as well. Linux IP Masquerading allows for this functionality even though these internal machines don't have an officially assigned IP address.<br>
>
> MASQ allows a set of machines to invisibly access the Internet via the MASQ gateway. To other machines on the Internet, the outgoing traffic will appear to be from the IP MASQ Linux server itself. In addition to the added functionality, IP Masquerade provides the foundation to create a HEAVILY secured networking environment. With a well-built firewall, breaking the security of a well-configured masquerading system and internal LAN should be considerably difficult to accomplish.

For further information, check this [great illustration](https://www.tldp.org/HOWTO/IP-Masquerade-HOWTO/ipmasq-background2.5.html)!

### binding the correct network interface

I did everything I can but still, I could not browse my website. What's wrong? Is the `localhost` the problem? I vaguely remember that sometimes `0.0.0.0` should be used instead of `127.0.0.1`.

According to [what is localhost and why should we use 127/8](http://www.tcpipguide.com/free/t_IPReservedPrivateandLoopbackAddresses.htm),

> The purpose of the loopback range is testing the TCP/IP protocol implementation on a host. Since the lower layers are short-circuited, sending to a loopback address allows the higher layers (IP and above, check [OSI model](https://en.wikipedia.org/wiki/OSI_model)) to be effectively tested without the chance of problems at the lower layers manifesting themselves. 127.0.0.1 is the address most commonly used for testing purposes.

I realize that the network interface `lo` won't accept traffic outside of the host! Do I have other network interfaces on this docker container?

```bash
# apt-get install iproute2
root@6dbee430cbd5:/guihao/Playground/jarvi-io# ip -4 a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
38: eth0@if39: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default  link-netnsid 0
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

`0.0.0.0` can solve this because, in the context of **server**, it means accept traffic on all interfaces, which means it will also accept traffic from device `eth0`.

[0]: https://guihao-liang.github.io/
[1]: https://github.com/github/personal-website
[2]: https://github.com/guihao-liang/guihao-liang.github.io
[3]: https://superuser.com/questions/949428/whats-the-difference-between-127-0-0-1-and-0-0-0-0
[4]: https://docs.docker.com/v17.09/engine/userguide/networking/
[5]: https://docs.docker.com/v17.09/engine/userguide/networking/default_network/binding/
