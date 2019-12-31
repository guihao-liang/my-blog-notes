---
layout: post
title:  "resume personal website guihao-liang.io with Jekyll"
date:  2019-12-29 12:27:42
categories: docker network jekyll
---

In order to save time, I like presenting conclusions at very beginning. ;-)

## Conclusion

I should've used this pre-configured [jekyll image](https://github.com/BretFisher/jekyll-serve) at very beginning to avoid all the troubles I met. And this image is quite easy to use:

```bash
cd dir/of/your/jekyll/site
# it will mount your current path into the containers /site,
# bundle install before running jekyll serve to
# serve it at http://<docker-host>:8080.
docker run -p 8080:4000 -v $(pwd):/site bretfisher/jekyll-serve
```

Bare metal installation of jekyll on a fresh Linux container is also viable. Just make sure the jekyll server is listening on [0.0.0.0, instead of 127.0.0.1](3).

If for security reason you cannot use `0.0.0.0`, you can use [ssh tunneling to forward traffic to docker](https://stackoverflow.com/questions/52321269/how-to-reach-docker-container-localhost-from-mac).

---

## motivation to resume my blog

I used to build jekyll from scratch to build [guihao-liang.io](0). I found the [personal static website template](1) is fairly easy to use. I decided to give a try. First I follow the [instructions](1) to fork a site for myself. Later on I realize that this site is based on Ruby, which I don't want to install on my local machine since I'm not a Ruby developer.

Oh well, there's one simplest way to avoid this hurdle is to just push to my [github.io repo](2) and test the correctiveness of the remotely generated site by github. If the site is not right, modify locally and push to the repository again. Repeat this process until I'm satisfied with the it. That's too troublesome for development. It's like there's no local test env, and all the dev work is verified from production directly. If user is upset, the further bug fix is on fire. That reminds me of some engineers that I've seen in my early career. Defnitely, I don't want to be one of of them.

Therefore, I realize I can have a docker container to run.

---

## Install jekyll on docker from scratch

At very beginning, I just pull a [Ubuntu docker image](https://hub.docker.com/_/ubuntu/) and run it as a container with interactive shell. At the same time, I realize I have to forward network ([binding ports](https://runnable.com/docker/binding-docker-ports)) from my Mac host to the container.

```bash
# -p host_port:container_port (mnemoic: forward from host to container)
docker run -it --name blog -p 5000:80 ubuntu:latest /bin/bash
```

In the pseudo-tty, I manually `apt-get installed` a bunch of things I need, such as git, sudo, [zsh](http://www.boekhoff.info/how-to-install-zsh-and-oh-my-zsh/), [oh-my-zsh](https://ohmyz.sh/), and [Jekyll](https://jekyllrb.com/docs/installation/ubuntu/). Normally, you don't need those tools to be installed since container is running without interacitve shells. Here, I plan to use this environment as long-term Linux box.

After installation is complete, I realize I don't have access to my [blog repo](https://github.com/guihao-liang/guihao-liang.github.io) from container instance to the host machine. The reason I need to have access to my host machine's file system is that I mainly edit my blogs on my Mac, where I have a nice GUI. As far as I googled, there's no way to mount the host file system at runtime. The worst case scenario is that I have to shut down the container and redo all the installations. At the time I planed to give up, I realize I can [commit](https://docs.docker.com/engine/reference/commandline/commit/) this container and save its state as an image!

```bash
# docker commit -a author -m "message"  <container_name> tag
docker commit -a guihaol -m "my blog image" blog gblog:test
```

Neat, I don't have to start over. This time, I need pass `-v` to tell docker which path I want to mount to container in order to [share data with host](https://www.digitalocean.com/community/tutorials/how-to-share-data-between-the-docker-container-and-the-host).

```bash
# -v <path_of_host>:<mount_path_of_container>
docker run -it --name blog -p 5000:80 -v /path/to/blog:/blog gblog:test /bin/zsh
```

The command is mounting local blog repo on my Mac host to the container's `/blog`. What I have to do next is to `cd` to `/blog` in the container, and follow [instructions](2) to start Jekyll:

```bash
gem install jekyll bundler
bundle install
bundle exec jekyll serve -p 5000
...

Server address: http://127.0.0.1:4000
Server running... press ctrl-c to stop.

```

I feel very excited to see my blog running locally! I provided `localhost:5000` to my browser, hitting enter key with extreme excitement, and I got `This site can’t be reached`. OMG! That means my network binding is not rignt! How dare you!

### container network binding

I already bind the ports from host to container, and why the container cannot be reached?

Since I used docker-for-Mac client, I checked [networking features](4) documentation. It shows that I'm using the right command `-p` to do the port-binding to the host, which creates a firewall rule which maps a container port to a port on the Docker host.

What this [hello world](https://github.com/karthequian/docker-helloworld/blob/a28dd5245c9347905330d0b16c36ddee41af76e1/Dockerfile#L45) does is just use docker command `EXPOSE 80`, which should be the same thing as `-p`.

Under the hood, it forwards the port from host to container. And we can [get the port mappings](https://stackoverflow.com/questions/32444612/how-to-get-the-mapped-port-on-host-from-a-docker-container),

```bash
docker port container_id
```

### ping container using its IP

Okay, it's mysterious, probably I should find the ip address to ping that `ip:port` directly. Hold, on, the doc also says:

> Q: I cannot ping my containers</br>
> A: Docker Desktop for Mac can’t route traffic to containers.</br>
> Q: Per-container IP addressing is not possible</br>
> A: The docker (Linux) bridge network is not reachable from the macOS host.</br>

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

I was told if I use docker command option `--net=host`, the container can share the network with my host, which means the container ip address should be an address from the host and reachable. Referenced from the [doc](https://docs.docker.com/network/host/):

> If you use the host network mode for a container, that container’s network stack is not isolated from the Docker host (the container shares the host’s networking namespace), and the container does not get its own IP-address allocated. For instance, if you run a container which binds to port 80 and you use host networking, the container’s application is available on port 80 on the host’s IP address.

That should work but it didn't. When I continued to read the doc, it says **host networking driver only works on Linux host**. Sign.

### bridge network

![docker mac desktop](https://docs.docker.com/docker-for-mac/images/docker-for-mac-install.png)

Bridge network is [used by default by docker](https://docs.docker.com/network/), but what is bridge network? And [how container connects to host through networking](https://www.docker.com/blog/understanding-docker-networking-drivers-use-cases/)?

![docker linux bridge](http://img.scoop.it/bmExZyvGWidultcwx9hCb7nTzqrqzN7Y9aBZTaXoQ8Q=)

The intial guess of mine is that the [bridge network](https://en.wikipedia.org/wiki/Bridging_(networking)) device is on link layer and aims to connect a group of hosts into a single local-area network (LAN). Based on my learning of computer networks, each IP node in the LAN maintains a [ARP](https://en.wikipedia.org/wiki/Address_Resolution_Protocol) table, which is the IP/MAC mappings. Within the LAN, network packets (or frames) are routed according to the host's MAC address. The `bridge` sounds like the switch device, which works at link layer and only needs to know MAC/interface mappings in order to direct the packets to right places.

> Containers connected to the default bridge network can communicate with each other by IP address. Docker does not support automatic service discovery on the default bridge network. If you want containers to be able to resolve IP addresses by container name, you should use user-defined networks instead.

Well, we cannot use `hostname` is probably due to the fact that no `dns` is bundled with this `bridge` network. We can only rely on the host's name resolution table,

```bash
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
# running as root
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

Surprisingly, `172.17.0.1` is the IP for the `bridge` device. The `bridge` network serves like a gateway **router**, which works at layer 4, instead of the layer 3 **switch**. The output of `arp -a` in the container doesn't have other hosts' IP/MAC mappings but only the gateway's. I reckon the topology of this bridge network is star-shaped, where all hosts connect to the gateway router and this gateway router routes the traffic in and out. To conlude, there's no LAN.

Great! If we can talk to this gateway, we might be able to ping the container with its IP!

Continue from this [doc](https://docs.docker.com/docker-for-mac/networking/):

> Q: There is no docker0 bridge on macOS</br>
> A: Because of the way networking is implemented in Docker Desktop for Mac, you cannot see a docker0 interface on the host. This interface is actually within the virtual machine.

`docker0` is the network bridge driver that is used by the container, which we've seen above, and it's not visable by the host.

Searching from StackOverflow, [xhyve vm inside Docker for Mac hasn't no Network Adapter](https://stackoverflow.com/questions/41819391/can-not-ping-docker-in-macos) and the `bridge` device runs inside of VM on Mac. That's a bummer.

The question is, if we cannot ping the ip, how docker can accept traffic? Well, similar to ssh tunneling, the docker listens on all network interfaces and forward the traffic to the VM, and the VM passes the traffic to the `bridge` network.

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

https://www.tldp.org/HOWTO/IP-Masquerade-HOWTO/ipmasq-background2.5.html
[masquerade](https://www.tldp.org/HOWTO/IP-Masquerade-HOWTO/ipmasq-background2.1.html)

### binding the correct network interface

I did everything I can but still I could not browse my website. What's wrong? Is the `localhost` the problem? I vaguely remember that sometimes `0.0.0.0` should be used instead of `127.0.0.1`.

After reading this post, [what is localhost and why should we use 127/8](http://www.tcpipguide.com/free/t_IPReservedPrivateandLoopbackAddresses.htm)

> The purpose of the loopback range is testing of the TCP/IP protocol implementation on a host. Since the lower layers are short-circuited, sending to a loopback address allows the higher layers (IP and above, check [OSI model](https://en.wikipedia.org/wiki/OSI_model)) to be effectively tested without the chance of problems at the lower layers manifesting themselves. 127.0.0.1 is the address most commonly used for testing purposes.

I realize that network interface `lo` won't accept traffic outside of the host! Do I have other network interfaces on this docker container?

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

`0.0.0.0` can solve this because, in the context of server, it means accept traffic on all interfaces, which means it will also accept traffic from device `eth0`.

[0]: https://guihao-liang.github.io/
[1]: https://github.com/github/personal-website
[2]: https://github.com/guihao-liang/guihao-liang.github.io
[3]: https://superuser.com/questions/949428/whats-the-difference-between-127-0-0-1-and-0-0-0-0
[4]: https://docs.docker.com/docker-for-mac/networking/