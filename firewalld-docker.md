# 解决firewalld无法限制docker发布的端口

### 问题描述
你的主机打开了防火墙，并且没有允许某个端口（这里假设是8080），你可能以为这个限制对主机上运行docker容器也生效。但是如果某个docker容器使用了`端口绑定`(也就是 -p 8080:80)，实际上依然可以从外部访问到这个8080端口，
这通常是在我们的预期之外，存在一定的[安全隐患](https://www.reddit.com/r/selfhosted/comments/1cv2l3q/security_psa_for_anyone_using_docker_on_a)。

### 问题原因
可以查看[docker官方说明](https://docs.docker.com/engine/network/packet-filtering-firewalls/#integration-with-firewalld)

TL;DR 总结来说就是docker为了实现端口转发和容器间通信，会自己插入一些iptables规则，默认允许所有流量通过。
如果你安装了firewalld，还会新建一个docker zone，同样默认允许所有流量通过。相当于绕过了你设置的防火墙规则，在你的防火墙上‘打了个洞’。

### 解决思路
如果你的linux版本还在使用iptables，可以使用[ufw-docker](https://github.com/chaifeng/ufw-docker)里面介绍的方法来限制docker暴露的接口。
但是目前很多新的linux发行版本，已经不使用iptables，而转向nftables，比如我在使用的Debian，从Debian 10开始就默认使用 nftables。

``` bash
sudo iptables --version
```

运行以上命令，如果输出 `iptables vx.x.x (nf_tables)`，说明你的linux版本已经默认使用的是nftables。
当然你可以重新安装回旧的iptables工具，但这显然不是一个优雅的方法。
基于以上原因，我们需要通过配置 nftables 来管理和限制 docker 暴露的端口。

### 详细过程
如果你的系统也使用 [firewalld](https://firewalld.org) 来管理系统防火墙(nftables)的规则，可以按照以下流程设置。
1. 添加以下配置，禁用docker iptables
   ```
   # /etc/docker/daemon.json
   {
       "iptables": false
   }
   ```
   这个设置告诉 docker 不要再添加任何防火墙规则。修改后，必须重启系统，才能完全清除docker设置的iptables规则。

   ⚠️ 注意：重启后所有的 docker 容器就会暂时无法连接网络，如果你需要通过某个 docker 容器来远程连接到你的主机，请回到本地环境后再操作。

2. 添加 docker zone 到firewalld
   ```
   sudo firewall-cmd --permanent --zone docker --add-source 172.17.0.0/16
   ```
   这里的`172.17.0.0/16`是docker默认的子网范围

3. 允许所有docker容器连接外网
   ```
   sudo firewall-cmd --permanent --new-policy docker-outbound
   sudo firewall-cmd --permanent --policy docker-outbound --add-ingress-zone docker
   sudo firewall-cmd --permanent --policy docker-outbound --add-egress-zone ANY
   sudo firewall-cmd --permanent --policy docker-outbound --set-target ACCEPT
   sudo firewall-cmd --permanent --policy docker-outbound --add-masquerade
   ```
   以上代码新建了一个`docker-outbound` policy，允许 docker zone的所有出口流量(ACCEPT)，也就是允许 docker zone内的所有容器连接外网，也包括容器与容器间的连接。

4. 入口流量转发到指定 docker 端口
   ```
   sudo firewall-cmd --permanent --new-policy dockerFwdPort
   sudo firewall-cmd --permanent --policy dockerFwdPort --add-ingress-zone ANY
   sudo firewall-cmd --permanent --policy dockerFwdPort --add-egress-zone HOST
   ```
   上面代码新建了一个名为`dockerFwdPort`的 policy，这个 policy 负责接收入口流量，并转发到指定的docker容器端口。

   接下来指定端口转发规则：
   ```
   sudo firewall-cmd --permanent --policy dockerFwdPort --add-forward-port port=8080:proto=tcp:toport=80:toaddr=172.17.0.2
   ```
   上面的代码指定了一条端口转发规则，把主机 `8080` 端口的入口流量，转发到了 `172.17.0.2` 这个docker容器的 `80` 端口。

   也可以通过 firewalld 的 rich-rule 来指定端口转发规则，实现更精细的管控：
   ```
   sudo firewall-cmd --permanent --zone public --add-rich-rule 'rule family="ipv4" source address="192.168.1.0/24" forward-port port="53" protocol="udp" to-port="53" to-addr="172.18.0.2"'
   ```
   假设我有一个DNS服务器运行在 `172.18.0.2` 容器上并绑定了主机的`53`端口，上面代码把`53`端口udp流量转发到了容器`53`端口上，但是限制了仅 `192.168.1.0/24` 来源可以访问。

5. 固定容器的ip
   你可能发现了，上面的端口转发规则，需要先知道端口转发的目的容器的ip(也就是 172.18.0.2)，但是通常容器被删除并重新启动后，ip就会变化。

   这个问题可以通过直接给容器指定静态ip解决：
   ```
   sudo docker run -it --name some_container --ip 172.18.0.2
   ```
   或者通过 compose.yml 指定
   ```
   networks:
     some_container_default:
       ipv4_address: 172.18.0.2
   ```

6. 最后别忘了重新加载使规则生效
   ```
   sudo firewall-cmd --reload
   ```

### 引用
[Strict Filtering of Docker Containers](https://firewalld.org/2024/04/strictly-filtering-docker-containers)

[stackexchange](https://unix.stackexchange.com/a/793249)
   
