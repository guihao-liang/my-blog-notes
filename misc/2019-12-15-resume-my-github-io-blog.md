---
layout: post
title:  "CPP Self Move Assignment"
date:   2018-11-25 15:22:02 -0800
categories: lang cpp
---

## motivation to resume my blog

I used to build jekyll from scratch to build [guihao-liang.io](0). I found the [personal static website template](1) is fairly easy to use. I decided to give a try. First I follow the [instructions](1) to fork a site for myself. Later on I realize that this site is based on Ruby, which I don't want to install on my local machine since I'm not a Ruby developer.

Oh well, there's one simplest way to avoid this hurdle is to just push to my [github.io repo](2) and test the correctiveness of the remotely generated site by github. If the site is not right, modify locally and push to the repository again. Repeat this process until I'm satisfied with the it. That's too troublesome for development. It's like there's no local test env, and all the dev work is verified from production directly. If user is upset, the further bug fix is on fire. That reminds me of some engineers that I've seen in my early career. Defnitely, I don't want to be one of of them.

Therefore, I realize I can have a docker container to run

## Install jekyll on docker

--net=host

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

Since I use docker-for-Mac client, I checked [networking features](https://docs.docker.com/docker-for-mac/networking/) documentation. It shows that I'm using the right command `-p` to do the port-binding. What this [hello world](https://github.com/karthequian/docker-helloworld/blob/a28dd5245c9347905330d0b16c36ddee41af76e1/Dockerfile#L45) does is just use docker command `EXPOSE 80`, which should be the same thing as `-p`.

Okay, it's mysterious, probably I should find the ip address to ping that `ip:port` directly. Hold, on, the doc also says:

> I cannot ping my containers</br>
> Docker Desktop for Mac can’t route traffic to containers.</br>
> Per-container IP addressing is not possible</br>
> The docker (Linux) bridge network is not reachable from the macOS host.</br>

From StackOverflow, the reason for this behavior is that [xhyve vm inside Docker for Mac hasn't no Network Adapter](https://stackoverflow.com/questions/41819391/can-not-ping-docker-in-macos). I'm not an expert in networks so I'd like to give up routing traffic to container's IP.



### Conclusion

I should've used this pre-configured [jekyll image](https://github.com/BretFisher/jekyll-serve) at very beginning to avoid all the troubles I met. And this image is quite easy to use:

```bash
cd dir/of/your/jekyll/site
# it will mount your current path into the containers /site,
# bundle install before running jekyll serve to
# serve it at http://<docker-host>:8080.
docker run -p 8080:4000 -v $(pwd):/site bretfisher/jekyll-serve
```


[0]: https://guihao-liang.github.io/
[1]: https://github.com/github/personal-website
[2]: https://github.com/guihao-liang/guihao-liang.github.io