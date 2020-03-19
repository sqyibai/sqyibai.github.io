## 安装
**安装python 3.6+**<br/>
ubuntu 18已经自带python3.6了,下面是ubuntu 16.04上的安装方法.

    sudo add-apt-repository ppa:deadsnakes/ppa
    sudo apt install python3.6
    curl https://bootstrap.pypa.io/get-pip.py | sudo python3.6
deadsnakes这个源可以支持到python3.9，所以上面的3.6可以替换成你想安装的版本

**安装paramiko库**<br/>

sudo pip3 install paramiko

**安装AT**<br/>
把解压缩得到的AT文件夹放在你想安装的位置，然后运行:

 python3 PATH_TO_AT/init.py
一路会有提示，一般回车即可。

安装完成之后还不能立刻运行命令，因为AT的安装原理是把自己的目录加入到系统路径，要等到下一次重启才生效，在此之前，可以用sudo来运行(sudo路径是立刻生效的)。

在非ubuntu系统上，"添加系统路径"这一步会失败，需要你手动把AT目录加入到你的系统PATH里。

## 本地命令
*trace* <br/>
跟踪打印一个软链接

version 
查看系统上已安装的某程序的全部版本
eg.
version python
version python3

mklink 
命令格式: mklink symlink target
target可以写成绝对路径或相对路径，当写成相对路径时，表示从符号文件的位置走到target。
eg.
mklink /usr/bin/xx  /usr/bin/python3.6
mklink  /usr/bin/yy ./python3.6
xx和yy指向的是同一个文件.


————AT命令集
AT命令主要提供了一系列基于ssh的远程操作
命令格式: AT operation arg1 arg2 ...


AT assign-priv-key user -kname
分配给user一个私钥，私钥取自name指定的KEY

user 要分配至的用户
-k   指定私钥的来源 
name可以是一个user name或KEY name,例如:
AT assign-priv-key jack -kBen       #把用户Ben的私钥赋给jack.
AT assign-priv-key jack -kshared    #把名为shared的KEY的私钥赋给jack.
KEY是AT用于管理密钥的一个概念,一个KEY由一个公钥和一个私钥组成，以一个包含id_rsa和id_rsa.pub的目录形式存放。所以Ben的~/.ssh目录也是一个KEY，只不过它需要用用户名指代。shared KEY是AT自动生成的，位于/root/.AT/keys目录，目前AT只支持这一个"具名"KEY. 
shared KEY是AT的缺省KEY,上面第二句命令可以写成:
AT assign-priv-key jack

分配私钥的过程是硬拷贝，除了拷贝id_rsa,还会把id_rsa.pub也拷贝过去。因为密钥一旦分配出去，AT就不知道这个密钥是从哪儿来的了，所以赶紧把公钥也拷贝过去，以免要用的时候无从找起。

AT assign-pub-key user@server -kname [--password]
分配公钥是把本地KEY的id_rsa.pub追加到远程服务器user账号下的authorized_keys文件。

AT的远程操作都包含一个自动登录的过程，自动登录时会依次尝试 当前用户，root用户(如果有权限的话)的密钥。
自动登录功能作为一项基础设施提供给每个操作，但具体登录哪个远程用户是由每个操作自己自己决定的,通常是root用户。
user@server这种格式可能会给你造成困扰，因为它的习惯上的含义是"指定以user身份登录server"，但在AT命令里里不是。就像前面所说的，大多数AT操作都有自己预设的登录用户作为远程操作者，它们不接受指定。这个前导user字段会被各个操作以自己的方式来解释。像比，assign-pub-key把这个字段解释成"要分配至的用户"。而add-remote-user则把这个字段解释成"要添加的用户"。

--password用来强制密码登录，这个选项主要是给assign-pub-key用，因为它面对的常常是一台崭新的服务器。

AT --config server
配置服务器信息, 这个命令调用外部编辑器打开一个叫server.json的文件
AT目前不接受直接在命令行输入ip,所以操作一台远程服务前，要先编辑这个文件。
初始状态的server.json大概长这样:
{
    "comment":"",
	"servers":
	[
		{
			"ip":"127.0.0.1"
		},

		{
			"name":"centos7",
			"ip":"0.0.0.0",
			"ssh_port":22
		}
	]
}
服务器列表以json数组的形式存储在'servers'字段,第一个服务器是最简单的形式，第二个是完整的格式。缺省情况下，ssh_port是22,name是s0,s1,s2.. ，即 "s" + indexof(server).
servers列表的第一个位置保留给本机，添加服务器时需要从第二个开始.

下面是一个实例，演示怎么通过assign操作实现密钥登录:
mz@bogon:~$ mv ~/.ssh ~/.ssh2       #我的工作目录已经有私钥了，先挪开
mz@bogon:~$ sudo AT assign-priv-key mz
key source: /root/.AT/keys/shared/
copy /root/.AT/keys/shared/id_rsa to /home/mz/.ssh/id_rsa ...
copy /root/.AT/keys/shared/id_rsa.pub to /home/mz/.ssh/id_rsa.pub ...

mz@bogon:~$ AT --config server
mz@bogon:~$ cat ~/.AT/server.json
{
    "comment":"",
	"servers":
	[
		{
			"ip":"127.0.0.1"
		},

		{
			"name":"centos7",
			"ip":"45.77.178.195",
			"ssh_port":22
		}
	]
}
	
mz@bogon:~$ sudo AT assign-pub-key root@centos7 --password
key source: /root/.AT/keys/shared/
发送远程命令: id -u root > /dev/null
login root@45.77.178.195, password:
创建socket: 45.77.178.195:22   [success]
发送远程命令: mkdir -p /root/.ssh
发送远程命令: touch /root/.ssh/authorized_keys
/root/.ssh/.authorized_keys.cook ===> /root/.ssh/authorized_keys
发送远程命令: chown -R `id -u root` /root/.ssh
发送远程命令: chmod 700 /root/.ssh
发送远程命令: chmod 600 /root/.ssh/.authorized_keys.cook
发送远程命令: mv '/root/.ssh/.authorized_keys.cook' '/root/.ssh/authorized_keys'

mz@bogon:~$ AT ssh2 root@centos7
创建socket: 45.77.178.195:22   [success]
login root@45.77.178.195, using '/home/mz/.ssh/id_rsa' ... [success]
root@centos7 $ 

最后成功用ssh2连上了服务器，ssh2是AT自带的一个测试命令，有时候openssh连不上，这个命令能连上.


其余的操作清单如下:
AT add-remote-user  user@server
添加一个远程用户,并创建家目录; 指定登录shell为bash; 添加进sudo用户组

AT del-remote-user user@server
AT list-remote-users @server

AT set-password user@server 设置用户口令

AT enable-password  @server
AT disable-password @server
开启/禁用密码登录

AT enable-pubkey  @server
AT enable-pubkey  @server
开启/禁用公钥登录

AT alt-ssh-port @server newport 修改ssh端口

AT init @server --password
初始化一台服务器,包括:
关闭selinux(针对redhat系)
给远程root用户分配公钥(公钥来自当前用户名下)
开启公钥登录
禁用密码登录


————scp2
scp2基本目的是为了利用上server.json里的配置，以便在命令行里用服务器名字代替ip.
这个工具在拷贝行为上，跟unix的传统风格有较大差异:
1, 复制一个目录，unix用-r,scp2用-d.
2, unix系的拷贝工具，像cp, scp,当它们发现复制目标是已存在的目录时，会自动的把拷贝对象扔到那个目录．例如:
cp .vim ~/.vim -r
复制的结果是，生成~/.vim/.vim这样一个拷贝．
在scp2下这样做会报错:
scp2 .vim ~/.vim -d
[error] file or directory exists: /home/kongque/.vim  [copy conflicts]
这是一个保护性错误，因为scp2是支持目录的对拷(说对拷其实不对，确切说是，当复制目标已经存在时，使用拷贝策略来解决冲突)的，但这种行为又很危险，所以需要你加上拷贝策略来确认. 
scp2有两个拷贝策略:
-a 全拷贝
-x 差异拷贝
scp2 .vim ~/.vim -da    #相当于用当前目录下的.vim替换掉~/.vim
scp2 .vim ~/.vim -dx    #跳过完全相同的文件
-x跟-a的区别，仅仅在于它可能会节省实际拷贝的工作量，从最终的拷贝结果来看，两者都是生成一个跟src一模一样的dest.请务必记住这条原则，这是拷贝的原始语意，否则这种伴随隐式重命名的拷贝，很容易让脑子浆糊．
3, 如果你需要往一个目录里拷贝，在目录末尾加上/
scp2 .vim  ~/.vim/

scp2底层包装的是linux的rsync命令，所以它提供了一部分rsync的特性,其中主要包含5个diff属性:
(下面的介绍约定本地目录为src,远程目录为dest)
-n 本地新增的文件(new added)
-u 本地缺少的文件(本地删除，也可能是服务器端有添加行为) 这个选项用字母u是取它的形状跟n相反,下面的w和m也是一样
-m 本地修改过的文件(modified)
-w 本地比远程旧的文件(可能是服务器端发生过编辑行为)
-e 完全相同的文件
numwe加起来是一个diff的全集，这些选项之间组合，可以做一些常见的操作:
scp2 -n src dest -d  复制src里新增的文件到服务器
scp2 -u src dest -d "复制"src里缺少的文件,实际上是往远程递交删除操作
scp2 -m src dest -d 复制src里修改过的文件到服务器
scp2 -num src dest -d 这有点儿像git的一次commit changes
scp2 -numw src dest -d 等同于上面的-x操作,差异复制
有些选项(主要是-w和-e)或选项组合是不支持的，像比单用一个w:
scp2 -w src dest -d
这样会报错，因为rsync提供的选项没有办法组合出这个逻辑．

这5个选项现在临时放在scp2里，将来可能会挪到别的命令里．
