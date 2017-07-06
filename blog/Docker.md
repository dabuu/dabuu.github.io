## Ubuntu 安装Docker ##
> 检查Ubuntu内核版本， 需要在3.10+， 越新越好
uname -a
sudo apt-get install -y --install-recommends linux-generic-lts-xenial

=== 重启机器。。。===

为了确认所下载软件包的合法性，需要添加 Docker 官方软件源的 GPG 密钥。
$ sudo apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D

用下面的命令将 APT 源添加到  source.list  （将其中的  <REPO>  替换为上表
的值）：
$ echo "<REPO>" | sudo tee /etc/apt/sources.list.d/docker.list
【dabu】: 
我一开始在测试机中使用的 repo 是 ： Xenial 16.04(LTS) deb https://apt.dockerproject.org/repo ubuntu-xenial main， 但运行install docker的时候会报错， 应该是 host的版本 低于 docker中ubuntu的版本
之后将docker.list 中换成 "deb https://apt.dockerproject.org/repo ubuntu-trusty main"


添加成功后，更新 apt 软件包缓存。
$ sudo apt-get update

安装Docker：
sudo apt-get install docker-engine

查看 Docker：
dabuwang@ubuntu:~$ sudo service docker start
start: Job is already running: docker
dabuwang@ubuntu:~$ service docker status
status: Unknown job: docker
dabuwang@ubuntu:~$ sudo service docker status
docker start/running, process 1562
dabuwang@ubuntu:~$ 
注意：用su 运行不了 docker，可以使用 “su -”
https://stackoverflow.com/questions/26137834/starting-docker-as-daemon-on-ubuntu

建立  docker  组：
$ sudo groupadd docker
将当前用户加入  docker  组：
$ sudo usermod -aG docker $USER
[dabu]: 如果没有讲当前用户加到 docker的组中， 在运行 docker pull 命令时 会出现 permission denied的情况
https://techoverflow.net/2017/03/01/solving-docker-permission-denied-while-trying-to-connect-to-the-docker-daemon-socket/

## 运行 docker ##

本地的镜像文件都存放在哪里？
答：与 Docker 相关的本地资源都存放在  /var/lib/docker/  目录下，以 aufs 文件系统为例，其中 container 目录存放容器信息，graph 目录存放镜像信息，aufs目录下存放具体的镜像层文件。

有了镜像后，我们就可以以这个镜像为基础启动一个容器来运行。以上面的ubuntu:14.04  为例，如果我们打算启动里面的  bash  并且进行交互式操作的话，可以执行下面的命令。
$ docker run -it --rm ubuntu:14.04 bash
root@e7009c6ce357:/# cat /etc/os-release
NAME="Ubuntu"
VERSION="14.04.5 LTS, Trusty Tahr"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 14.04.5 LTS"
VERSION_ID="14.04"
HOME_URL="http://www.ubuntu.com/"
SUPPORT_URL="http://help.ubuntu.com/"
BUG_REPORT_URL="http://bugs.launchpad.net/ubuntu/"
root@e7009c6ce357:/# exit
exit
$
docker run  就是运行容器的命令，具体格式我们会在后面的章节讲解，我们这里简要的说明一下上面用到的参数。
-it  ：这是两个参数，一个是  -i  ：交互式操作，一个是  -t  终端。我们这里打算进入  bash  执行一些命令并查看返回结果，因此我们需要交互式终端。
--rm  ：这个参数是说容器退出后随之将其删除。默认情况下，为了排障需求，退出的容器并不会立即删除，除非手动  docker rm  。我们这里只是随便执行个命令，看看结果，不需要排障和保留结果，因此使用  --rm  可以避免浪费空间。
ubuntu:14.04  ：这是指用  ubuntu:14.04  镜像为基础来启动容器。
bash  ：放在镜像名后的是命令，这里我们希望有个交互式 Shell，因此用的是  bash  。
进入容器后，我们可以在 Shell 下操作，执行任何所需的命令。这里，我们执行了cat /etc/os-release  ，这是 Linux 用的查看当前系统版本的命令，从返回的
结果可以看到容器内是  Ubuntu 14.04.5 LTS  系统。
最后我们通过  exit  退出了这个容器。
【dabu】：在bash中可以完成基本的操作， 我可以列出来 python的所在路径， 但没有python 解释器。
root@6fa93948f0a8:/usr/share# whereis python
python: /usr/bin/python3.4 /usr/bin/python3.4m /etc/python3.4 /usr/lib/python3.4 /usr/lib/python2.7 /usr/local/lib/python3.4
root@6fa93948f0a8:/usr/lib/python2.7/dist-packages# python
bash: python: command not found


##  利用 commit 理解镜像构成  ##
下载 nginx 镜像
> docker pull nginx

现在让我们以定制一个 Web 服务器为例子，来讲解镜像是如何构建的。
这条命令会用  nginx  镜像启动一个容器，命名为  webserver  ，并且映射了 80端口，这样我们可以用浏览器去访问这个  nginx  服务器。
如果是在 Linux 本机运行的 Docker，或者如果使用的是 Docker for Mac、Dockerfor Windows，那么可以直接访问：http://localhost；
> docker run --name webserver -d -p 80:80 nginx

我们希望改成欢迎 Docker 的文字，我们可以使用  docker exec  命令进入容器，修改其内容。
$ docker exec -it webserver bash
root@3729b97e8226:/# echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
root@3729b97e8226:/# exit
exit

我们修改了容器的文件，也就是改动了容器的**存储层**。我们可以通过  docker diff  命令看到具体的改动。
dabuwang@ubuntu:~$ docker diff webserver 
C /root
A /root/.bash_history
C /run
A /run/nginx.pid
C /usr
C /usr/share
C /usr/share/nginx
C /usr/share/nginx/html
C /usr/share/nginx/html/index.html
C /var
C /var/cache
C /var/cache/nginx
A /var/cache/nginx/client_temp
A /var/cache/nginx/fastcgi_temp
A /var/cache/nginx/proxy_temp
A /var/cache/nginx/scgi_temp
A /var/cache/nginx/uwsgi_temp

当我们运行一个容器的时候（如果不使用卷的话），我们做的任何文件修改都会被记录于容器存储层里。而 Docker 提供了一个  docker commit  命令，可以将容器的存储层保存下来成为镜像。换句话说，就是在原有镜像的基础上，再叠加上容器的存储层，并构成新的镜像。以后我们运行这个新镜像的时候，就会拥有原有容器最后的文件变化。
docker commit  的语法格式为：
docker commit [选项] <容器ID或容器名> [<仓库名>[:<标签>]]

我们可以用下面的命令将容器保存为镜像：
其中  --author  是指定修改的作者，而  --message  则是记录本次修改的内容。这点和  git  版本控制相似，不过这里这些信息可以省略留空。
$ docker commit \
	--author "Tao Wang <twang2218@gmail.com>" \
	--message "修改了默认网页" \
	webserver \
	nginx:v2
sha256:07e33465974800ce65751acc279adc6ed2dc5ed4e0838f8b86f0c87aa1795214
**慎用  docker commit 修改存储的镜像** 定制镜像的话 可以用dockerfile来完成。

 ##  dockerfile  ##