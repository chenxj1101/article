# vsftpd安装使用说明

`vsftpd`是在`linux`环境下，使用最多的`FTP`服务端软件。

`vsftpd`默认只能使用root用户运行。使用非`root`用户运行，需要在配置文件里设置`run_as_launching_user=YES`。

**官方强烈不推荐使用这种方式启动，会带来巨大的安全问题，并且会导致无法使用`chroot`技术来限制文件访问。**

以下安装配置均在`root`账号下进行。

## 安装====

```bash
yum install -y vsftpd
yum install -y db4 #设置虚拟账户的本地数据库文件用
```

也可以下载源码安装，解压之后直接`make & make install`即可安装。  
不提供`configure`文件，所以无法指定路径。

安装完成后会在相关目录生成文件，需要用到的如下：

```bash
/usr/sbin/vsftpd #vsftpd可执行文件

/etc/vsftpd/vsftpd.conf #主配置文件

/etc/pam.d/vsftpd #PAM认证文件

/etc/vsftpd.ftpusers #禁用使用VSFTPD的用户列表文件

/etc/vsftpd.user_list #禁止或允许使用VSFTPD的用户列表文件
```

## 配置

### vsftpd主要配置

先对默认配置文件进行备份，再进行配置。

```bash
cp /etc/vsftpd/vsftpd.conf /etc/vsftpd/vsftpd.conf.bak
```

配置文件具体配置如下：

```bash
anonymous_enable=NO #不允许匿名登录
local_enable=YES
write_enable=YES
local_umask=022  #上传文件权限补码，最终上传后的文件权限为  666-022=644
dirmessage_enable=YES
xferlog_enable=YES
connect_from_port_20=YES
xferlog_std_format=YES
ascii_upload_enable=YES
ascii_download_enable=YES
chroot_local_user=NO #虚拟账户配置下，在下面两个chroot配置后，这个参数必须为NO，否则登陆FTP后还可以访问其他目录！
chroot_list_enable=YES 
allow_writeable_chroot=YES
chroot_list_file=/home/ftp/config/chroot_list #指定不能离开家目录的用户列表文件，一行一个用户。使用此方法时必须chroot_local_user=NO。说明这个列表里面的用户登陆ftp后都只能访问其主目录，其他目录都不能访问！
listen=YES #监听IPV4
listen_ipv6=NO #监听IPV6，不能和上面的listen同时设置为YES

pam_service_name=vsftpd #指定PAM配置文件，即下面的/etc/pam.d/vsftpd文件要和这里指定的一致
userlist_enable=YES
tcp_wrappers=YES

virtual_use_local_privs=YES
guest_enable=YES #启用虚拟账户
guest_username=chenxj #将虚拟用户映射为本地chenxj用户（前提是local_enable=YES），更安全的做法是映射为nobody用户，因为nobody的权限最低

user_config_dir=/home/ftp/config/vuser_conf #指定不同虚拟用户配置文件的存放路径

listen_port=21 #监听的ftp端口，改成其他端口，会导致无法启动
pasv_min_port=40001 #分配给ftp账号的最小端口。被动模式下的配置
pasv_max_port=40100 #分配给ftp账号的最大端口。每个账号分配一个端口，即最大允许100个ftp账号连接
max_clients=150 #客户端的最大连接数
accept_timeout=5
connect_timeout=1
max_per_ip=5 #每个ip最大连接数
```

**以上配置有修改，需要重启vsftpd服务才能生效**。

### 虚拟账户设置

在自己决定的目录（如：`/home/ftp/config`）新建`vuser_passwd.txt`，奇数行为账号，偶数行为密码:

```bash
user1  #账号1
user1@2018 #账号1密码
user2 #账号2
user2@2018 #账号2密码
```

在此目录下，生成虚拟用户口令认证的db文件，这是本地数据库文件：

```bash
cd /home/ftp/config
db_load -T -t hash -f vuser_passwd.txt vuser_passwd.db
chmod 600 vuser_passwd.db #安全起见，将该文件的权限设置为root读写
```

同时在该目录下，新建`chroot_list`文件，即`/etc/vsftpd/vsftpd.conf`中的`chroot_list_file=`，并将虚拟账户的账号放在这个文件中。

```bash
cat chroot_list
user1
user2
```

然后新建`vuser_conf`目录，即`/etc/vsftpd/vsftpd.conf`中的`user_config_dir=`。用于存放每个虚拟账户的配置文件，虚拟账户的配置文件以虚拟账户的用户名命名。

```bash
mkdir vuser_conf
vim user1
# 配置如下：
local_root=/home/ftp/data/user1 #目录需已经存在，拥有者需为/etc/vsftpd/vsftpd.conf 中guest_username指定的用户
write_enable=YES
anon_umask=022
anon_world_readable_only=NO
anon_upload_enable=YES
anon_mkdir_write_enable=YES
anon_other_write_enable=YES
```

**以上配置有修改，不需要重启vsftpd，即时生效**。

### PAM认证

修改`/etc/pam.d/vsftpd`文件，注释掉原来的内容，在最后两行添加认证文件路径，如下所示：

```bash
#%PAM-1.0
#session    optional     pam_keyinit.so    force revoke
#auth       required	pam_listfile.so item=user sense=deny file=/etc/vsftpd/ftpusers onerr=succeed
#auth       required	pam_shells.so
#auth       include	password-auth
#account    include	password-auth
#session    required     pam_loginuid.so
#session    include	password-auth
auth	required	/lib64/security/pam_userdb.so	db=/home/ftp/config/vuser_passwd
account required	/lib64/security/pam_userdb.so db=/home/ftp/config/vuser_passwd
```

中间使用`Tab`键分隔。

## 启动

```bash
systemctl start vsftpd #启动
systemctl restart vsftpd #重启
systemctl stop vsftpd #停止
systemctl status vsftpd #查看状态
```

## 管理

写了`3`个小脚本来管理。

### 新增用户

```bash
sh /home/ftp/script/setFTP.sh <user> <passwd> <path>
```

代码如下：

```bash
#!/bin/sh

user=$1
passwd=$2
path=$3

echo $user >> /home/ftp/config/chroot_list
echo -e "$user\n$passwd" >> /home/ftp/config/vuser_passwd.txt
db_load -T -t hash -f /home/ftp/config/vuser_passwd.txt /home/ftp/config/vuser_passwd.db

mkdir $path
chown -R chenxj:chenxj $path
chmod 700 $path

echo -e "local_root=$path\nwrite_enable=YES\nanon_world_readable_only=NO\nanon_upload_enable=YES\nanon_mkdir_write_enable=YES\nanon_other_write_enable=YES" > /home/ftp/config/vuser_conf/$user
```

### 取消和恢复

如果希望某个路径关闭FTP，则直接修改其权限为`600`即可，则在配置文件中`local_root`指定为该路径的账号全都无法登陆。同理，恢复则将权限修改为`700`：

```bash
sh /home/ftp/script/cancelFTP.sh <path> #chmod 600 $path
sh /home/ftp/script/recoverFTP.sh <path> #chmod 700 $path
```
