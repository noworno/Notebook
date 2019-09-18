*2019-7-18*

---

## Ubuntu Linux下openssh-server配置及基本使用方法

### 一、安装和启动

```shell
apt install openssh-server		#安装openssh-server
service ssh start			#启动openssh-server
ps -e |grep sshd			#查看ssh服务是否启动  -e查看全部进程
service ssh status                      #查看ssh运行状态
```

设置开机启动，进入`/etc/rc.local`编辑配置

`vi /etc/rc.local`
在最后插入两行

`service ssh start`
`exit 0`
保存退出

这样即可在Ubuntu开机时自动启动ssh-server服务

### 二、登陆Linux服务器

```shell
ssh remote_username@remote_ip#用户名，命令执行后需要再输入密码
ssh remote_ip#没有指定用户名，命令执行后需要输入用户名和密码（有时默认用户名为root）
```

### 三、允许root登陆

进入`/etc/ssh/sshd_config`查找`PermitRootLogin`选项（可以利用`:/PermitRootLogin`的方法进行查找）

将这个选项后面的值（一般为`prohibit-password`）修改为`yes`

修改完成后保存退出，需要重启ssh服务：

`service ssh restart`

### 四、文件传输

#### windows下（需要使用putty/Xshell/MobaXterm任选其一）

方法一：

```shell
apt install lrzsz			#Linux安装文件上传下载工具lrzsz，不能传输大于4G的文件
rz					#弹出对话框，选择文件从windows下载文件
sz filename				#将filename文件发送到windows，弹出对话框，选择windows路径保存位置
```

方法二：
    将windows里的文件直接拖动到Xshell里面

方法三：
    Xshell中：新建文件传输（Ctrl+Alt+F）

#### Linux下

`scp`命令（个人理解为ssh+cp命令，功能与cp命令类似）

##### 1、从本地复制到远程

命令格式：

```shell
#以下两个指定了用户名，命令执行后需要再输入密码
scp local_file remote_username@remote_ip:remote_folder#仅指定了远程的目录，文件名字不变
scp local_file remote_username@remote_ip:remote_file#指定了文件名
#以下两个没有指定用户名，命令执行后需要输入用户名和密码
scp local_file remote_ip:remote_folder#仅指定了远程的目录，文件名字不变
scp local_file remote_ip:remote_file#指定了文件名
```

应用实例：

```shell
scp /home/ubuntu/music/1.mp3 root@192.168.100.100:/home/root/others/music
scp /home/ubuntu/music/1.mp3 root@192.168.100.100:/home/root/others/music/100.mp3
scp /home/ubuntu/music/1.mp3 192.168.100.100:/home/root/others/music
scp /home/ubuntu/music/1.mp3 192.168.100.100:/home/root/others/music/100.mp3
```

复制目录命令格式：

```shell
scp -r local_folder remote_username@remote_ip:remote_folder#指定了用户名，命令执行后需要再输入密码
scp -r local_folder remote_ip:remote_folder#没有指定用户名，命令执行后需要输入用户名和密码
```

应用实例：

```shell
scp -r /home/ubuntu/music/ root@192.168.100.100:/home/root/others/
scp -r /home/ubuntu/music/ 192.168.100.100:/home/root/others/
```

上面命令将本地 music 目录复制到远程 others 目录下。

##### 2、从远程复制到本地

从远程复制到本地，只要将从本地复制到远程的命令的后2个参数调换顺序即可，如下实例

应用实例：

```shell
scp root@192.168.100.100:/home/root/others/music /home/ubuntu/music/1.mp3
scp -r 192.168.100.100:/home/root/others/ /home/ubuntu/music/
```

说明

1.如果远程服务器防火墙有为`scp`命令设置了指定的端口，我们需要使用 `-P` 参数来设置命令的端口号，命令格式如下：

```shell
#scp 命令使用端口号 2222
scp -P 2222 remote_username@192.168.100.100:/home/root/others/music /home/ubuntu/music
```

2.如果使用普通用户（非root用户）使用ssh操作远程主机使用scp命令要确保使用的用户具有可读取远程服务器相应文件的权限，否则scp命令是无法起作用的。

### 五、免密登陆

Linux服务器上将`/etc/ssh/sshd_config`文件里面


这些选项前的注释去掉，保存退出

windows
    putty
使用puttygen.exe生成密钥（生成过程中多移动鼠标加快密钥生成速度）
将公钥内容全部复制到服务器的/root/.ssh/authorized_keys中
（authorized_keys名称是由/etc/ssh/sshd_config文件中AuthorizedKeysFile选项决定的）
将私钥保存到windows中（我这里保存到E盘中）
putty中设置：


    Session中填写对应的IP地址
    Connection->data->Auto-login username中填写要登陆的用户名
    Connection->SSH->Auth中选择保存的私钥（我这里保存到E盘中）


设置完成后就能进行免密登陆了

Linux

```shell
ssh remote_username@remote_ip
mkdir /root/.ssh
chmod 700 /root/.ssh
exit
ssh-keygen -t rsa
cd /root/.ssh
scp id_rsa.pub remote_username@remote_ip:/root/.ssh
ssh remote_username@remote_ip
cat id_rsa.pub >authorized_keys
exit
```

或

```shell
#Linux自带命令
ssh-keygen -t rsa
ssh-copy-id remote_username@remote_ip
```

---------------------
