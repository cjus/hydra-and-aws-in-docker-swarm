## Create Redis and ELK instances

Create two t2.small EC2 instances using Amazon ECS optimized image.

Setup Redis:

```shell
$ docker pull redis:3.0.7
$ sudo mkdir /data

Add this to /etc/rc.local

docker rm -f redis
docker run -d -p 6379:6379 --restart always -v /data:/data --name redis redis:3.0.7

Then:

$ sudo reboot
```

Setup ELK:

```shell
$ sudo docker pull elasticsearch:2
$ sudo docker pull kibana:4
$ sudo mkdir /data

Add this to /etc/rc.local

HOST=`ifconfig eth0 | grep "inet addr" | cut -d ':' -f 2 | cut -d ' ' -f 1`
docker rm -f elasticsearch
docker run -d -p 9200:9200 -p 9300:9300 --restart always --ulimit nofile=2048:2048 -v /data:/data --name elasticsearch elasticsearch:2
docker rm -f kibana
docker run -d -p 5601:5601 --restart always --add-host elasticsearch:$HOST -v /data:/data --name kibana kibana:4

Then:

$ sudo reboot
```
