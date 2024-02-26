---
typora-copy-images-to: Nessus
---

#### 下载

在官网：https://www.tenable.com/downloads/nessus?loginAttempted=true

选择对应的版本

#### 安装

在下载得到的deb文件所在目录执行

```
dpkg -i Nessus-8.10.0-debian6_amd64.deb
```

#### 启动

```
/etc/init.d/nessusd start
```

#### 登陆web界面

```
https://guet:8834/
https://localhost:8834/
```

#### 忘记密码

跳转到`Nessus`目录

```
cd /opt/nessus/sbin/
列出所有用户
sudo ./nessuscli lsuser
更新密码
sudo ./nessuscli chpasswd #要更新密码的用户名
等到提示 【New password: 】的时候输入新密码，需要输两次 
```

在安装`Nessus`过程中有可能会出现插件下载失败的问题，此时，我们可以使用离线下载的方式进行插件下载。

1、首先生成挑战码，在目录`/opt/nessus/sbin`下执行以下代码生成挑战码：

`./nessuscli fetch --challenge`

![image-20200408224630932](E:\Kali\Nessus\image-20200408224630932.png)

2、接着打开网页：https://plugins.nessus.org/v2/offline.php ，输入挑战码和激活码。

注：由于激活码只能使用一次，因此我们可在：https://www.tenable.com/products/nessus/activation-code再次获取激活码。

3、输入后点击“submit”按钮，将会调转到插件下载页面，如下图所示：

![image-20200408225028075](E:\Kali\Nessus\image-20200408225028075.png)

4、下载插件和注册码后（注意：由于是国外的网站，插件下载速度可能较慢），接着将该两个复制到`Nessus`的`/opt/nessus/sbin/`目录下，运行如下命令更新插件：

```
./nessuscli fetch --register-offline nessus.license
./nessuscli update all-2.0.tar.gz
```


或登陆Nessus → Setting → Software Update 右上角 Manual Software Update，选择Upload your own plugin archive，找到刚上传的all-2.0.tar.gz文件，等一段时间就升级好了。

5、插件升级后，使用命令`./nessusd`重启`Nessus`即可。

