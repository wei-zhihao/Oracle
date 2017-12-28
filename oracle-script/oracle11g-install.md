
## This script is in centos7 install Oracle 11g
### ALL the default password is oracle_2017

```
#!/bin/sh
#Oracle 11g R2 install
#The author weizhihao
#August 31, 2017

#1、安装Oracle所需的依赖包
sudo yum -y install  gcc gcc-c++ make binutils compat-libstdc++-33 glibc glibc-devel libaio libaio-devel libgcc libstdc++ libstdc++-devel unixODBC unixODBC-devel sysstat ksh

#2、创建用户和组

sudo  groupadd -g 200 oinstall  #添加oinstall组，组的id为200
sudo  groupadd -g 201 dba       #添加dba组，组的id为201
sudo  useradd -u 440 -g oinstall -G dba oracle  #添加用户oracle,并specified它的id为440
sudo echo oracle_2017 | passwd  --stdin oracle #输入oracle用户的密码

#3、创建目录并授权
sudo  mkdir -p /data1/oracle/app
sudo  chmod 755 /data1/oracle
sudo  chown oracle.oinstall -R /data1/oracle


#4、关闭SELINUX(阿里云缺省关闭)

sudo  sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' \/etc\/selinux\/config
sudo  setenforce 0 

#5、编辑内核

sudo  cp /etc/sysctl.conf /etc/sysctl.conf.bak

sudo echo "#oracel" >>/etc/sysctl.conf

sudo  /sbin/sysctl -a | grep sem | sed -n '1p' >>/etc/sysctl.conf
sudo  /sbin/sysctl -a | grep shm | sed -n '3,5p' >>/etc/sysctl.conf
sudo  /sbin/sysctl -a | grep file-max| sed -n '1p ' >>/etc/sysctl.conf
sudo /sbin/sysctl -a | grep ip_local_port_range| sed -n '1p ' >>/etc/sysctl.conf
sudo /sbin/sysctl -a | grep rmem_default | sed -n '1p ' >>/etc/sysctl.conf
sudo /sbin/sysctl -a | grep rmem_max | sed -n '1p ' >>/etc/sysctl.conf
sudo /sbin/sysctl -a | grep wmem_default | sed -n '1p ' >>/etc/sysctl.conf
sudo /sbin/sysctl -a | grep wmem_max | sed -n '1p ' >>/etc/sysctl.conf
sudo /sbin/sysctl -a | grep fs.aio-max-nr | sed -n '1p ' >>/etc/sysctl.conf

#保存后生效命令
sudo /sbin/sysctl -p

#6、配置环境变量

sudo cp /etc/profile /etc/profile.bak

sudo echo "#oracel" >>/etc/profile


sudo echo "if [ \$USER = "oracle" ]; then
        if [ \$SHELL = "/bin/ksh" ]; then
              ulimit -p 16384
              ulimit -n 65536
        else
              ulimit -u 16384 -n 65536
        fi
fi" >>/etc/profile


#7、设置进程数和最大会话数
sudo cp /etc/security/limits.conf /etc/security/limits.conf.bak
sudo echo "#oracle" >>/etc/security/limits.conf

sudo echo "oracle           soft    nproc           2047
oracle           hard    nproc           16384
oracle           soft    nofile          1024
oracle           hard    nofile          65536" >>/etc/security/limits.conf

#8、关联设置
#编辑文件/etc/pam.d/login 加入以下语句:
sudo cp /etc/pam.d/login  /etc/pam.d/login.bak
sudo sed -i '17a session    required     pam_limits.so' /etc/pam.d/login 

#9、oracle 环境配置

sudo cp /home/oracle/.bash_profile /home/oracle/.bash_profile.bak

ORACLE_BASE="export ORACLE_BASE\=/data1/oracle/app"
ORACLE_HOME="export ORACLE_HOME\=\$\ORACLE_BASE/product/11.2.0/dbhome_1"
ORACLE_PATH="export PATH\=\$\PATH:\$\ORACLE_HOME/bin"
ORACLE_SID="export ORACLE_SID\=orcl"
TNS_ADMIN="export TNS_ADMIN\=\$\ORACLE_HOME/network/admin"
LD_LIBRARY_PATH="export LD_LIBRARY_PATH\=\$\ORACLE_HOME/lib:/lib:/usr/li"

sudo su - oracle -c "echo $ORACLE_BASE" >>/home/oracle/.bash_profile
sudo su - oracle -c "echo $ORACLE_HOME" >>/home/oracle/.bash_profile
sudo su - oracle -c "echo $ORACLE_SID" >>/home/oracle/.bash_profile
sudo su - oracle -c "echo $ORACLE_PATH" >>/home/oracle/.bash_profile
sudo su - oracle -c "echo $TNS_ADMIN" >>/home/oracle/.bash_profile
sudo su - oracle -c "echo $LD_LIBRARY_PATH" >>/home/oracle/.bash_profile

sudo su - oracle -c "source ~/.bash_profile"

#10 解压程序

cd /data1/src

unzip linux.x64_11gR2_database_1of2.zip
unzip linux.x64_11gR2_database_2of2.zip

if [ -d database ]
 then 
rm -rf /data1/src/database/response/dbca.rsp
rm -rf /data1/src/database/response/db_install.rsp
cp dbca.rsp /data1/src/database/response/
cp db_install.rsp /data1/src/database/response/
else 
echo "database directory does not exist"
exit 1 
fi

#创建Oracle用户下应用 安装脚本
cd /home/oracle/
echo "#!/bin/sh" >>/home/oracle/install.sh
echo "cd /data1/src/database" >>/home/oracle/install.sh
echo "./runInstaller -silent -ignorePrereq -ignoreSysPrereqs -responseFile /data1/src/database/response/db_install.rsp" >>/home/oracle/install.sh
chmod +x /home/oracle/install.sh


##创建Oracle用户下 建库dbca 脚本

echo "#!/bin/sh" >> /home/oracle/dbca.sh
echo "cd /data1/src/database" >>/home/oracle/dbca.sh
echo "dbca -silent -responseFile /data1/src/database/response/dbca.rsp" >>/home/oracle/dbca.sh
chmod +x /home/oracle/dbca.sh

#lsn脚本

echo " #!/bin/sh " >>/home/oracle/lsn.sh
echo "lsnrctl start" >>/home/oracle/lsn.sh
chmod +x /home/oracle/lsn.sh

#启动安装脚本
su - oracle -s /bin/bash /home/oracle/install.sh
sleep 320
sh /data1/oracle/oraInventory/orainstRoot.sh >/dev/null 2>&1
sleep 2
sh /data1/oracle/app/product/11.2.0/dbhome_1/root.sh >/dev/null 2>&1
sleep 2
su - oracle -s /bin/bash /home/oracle/dbca.sh >/dev/null 2>&1
sleep 5
echo "Oracle installation is complete"
rm -rf /data1/src/database 
sleep 2
su - oracle -s /bin/bash /home/oracle/lsn.sh
sleep 3
exit 0

```













