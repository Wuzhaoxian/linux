# jumpserver部署

环境准备

```
systemctl stop firewalld
setenforce 0
```

1、安装Python3

```
yum -y install gcc gcc-c++ zlib-devel bzip2-devel openssl-devel  sqlite-devel readline-devel  libffi-devel
wget  https://www.python.org/ftp/python/3.6.5/Python-3.6.5.tgz
tar -xf Python-3.6.5.tgz
cd Python-3.6.5/
sed -ri 's/^#readline/readline/' Modules/Setup.dist
sed -ri 's/^#(SSL=)/\1/' Modules/Setup.dist
sed -ri 's/^#(_ssl)/\1/' Modules/Setup.dist 
sed -ri 's/^#([\t]*-DUSE)/\1/' Modules/Setup.dist 
sed -ri 's/^#([\t]*-L\$\(SSL\))/\1/' Modules/Setup.dist
./configure  --enable-shared
make -j 2 && make install
cat >> /etc/profile.d/python3_lib.sh <<EOF
export   LD_LIBRARY_PATH=\$LD_LIBRARY_PATH:/usr/local/lib
EOF
cat >> vim  /etc/ld.so.conf.d/python3.conf <<EOF
/usr/local/lib
EOF
ldconfig
source /etc/profile
```

2、建立 Python 虚拟环境

因为 CentOS 6/7 自带的是 Python2，而 Yum 等工具依赖原来的 Python，为了不扰乱原来的环境我们来使用 Python 虚拟环境

```
cd /opt
python3 -m venv py3
source /opt/py3/bin/activate
```

注：看到下面的提示符代表成功，以后运行 Jumpserver 都要先运行以上 source 命令，以下所有命令均在该虚拟环境中运行

(py3) [root@centos7-1 opt]#

3、安装jumpserver 1.0.0

```
cd /opt
mkdir jumpserver
git clone --depth=1 https://github.com/jumpserver/jumpserver.git && cd jumpserver && git checkout master
cd /opt/jumpserver/requirements
yum -y install $(cat rpm_requirements.txt)
```

4、安装Python库依赖

`pip install -r requirements.txt`

5、安装redis、mysql

```
yum -y install redis
systemctl start redis && systemctl enable redis
yum -y install mariadb mariadb-devel mariadb-server
systemctl start mariadb && systemctl enable mariadb
```

6、创库并修改配置文件

```
#随机24位数据库jumpserver密码。可修改

DB_PASSWORD=`cat /dev/urandom | tr -dc A-Za-z0-9 | head -c 24`
SECRET_KEY=`cat /dev/urandom | tr -dc A-Za-z0-9 | head -c 50`
echo "SECRET_KEY=$SECRET_KEY" >> ~/.bashrc
BOOTSTRAP_TOKEN=`cat /dev/urandom | tr -dc A-Za-z0-9 | head -c 16`
echo "BOOTSTRAP_TOKEN=$BOOTSTRAP_TOKEN" >> ~/.bashrc
mysql -uroot -e "create database jumpserver default charset 'utf8';grant all on jumpserver.* to 'jumpserver'@'127.0.0.1' identified by '$DB_PASSWORD';flush privileges;"
cp /opt/jumpserver/config_example.yml /opt/jumpserver/config.yml
sed -i "s/SECRET_KEY:/SECRET_KEY: $SECRET_KEY/g" /opt/jumpserver/config.yml
sed -i "s/BOOTSTRAP_TOKEN:/BOOTSTRAP_TOKEN: $BOOTSTRAP_TOKEN/g" /opt/jumpserver/config.yml
sed -i "s/# DEBUG: true/DEBUG: false/g" /opt/jumpserver/config.yml
sed -i "s/# LOG_LEVEL: DEBUG/LOG_LEVEL: ERROR/g" /opt/jumpserver/config.yml
sed -i "s/# SESSION_EXPIRE_AT_BROWSER_CLOSE: false/SESSION_EXPIRE_AT_BROWSER_CLOSE: true/g" /opt/jumpserver/config.yml
sed -i "s/DB_PASSWORD: /DB_PASSWORD: $DB_PASSWORD/g" /opt/jumpserver/config.yml
```

7、生成数据库表结构和初始化数据

```
cd /opt/jumpserver/utils
bash make_migrations.sh
```

8、运行

```
cd /opt/jumpserver
./jms start all
#./jms stop|start|status|restart all 
```

9、浏览器访问http://localhostIP:8080

注意：

① 第一次运行时可能报错，(这里只是 Jumpserver, 没有 Web Terminal，所以访问 Web Terminal 会报错)

② 终止程序，再次执行，就可以登录了

(py3) [root@centos7-1 jumpserver]# ./jms start all 

账号: admin 密码: admin

登录成功

![1560941368841](C:\Users\53533\AppData\Local\Temp\1560941368841.png)