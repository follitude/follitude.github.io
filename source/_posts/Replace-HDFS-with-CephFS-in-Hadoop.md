---
title: Replace HDFS with CephFS in Hadoop (Stand-alone)
---
##### 环境
```
lsb_release -d
Ubuntu 16.04.3 LTS

java -version
openjdk version "1.8.0_131"

ant -version
Ant：Apache Ant(TM) version 1.9.6
```

##### 下载源码
###### Hadoop
```
git clone https://github.com/apache/hadoop.git
```

###### Ceph
```
git clone https://github.com/ceph/ceph.git
```

##### 分支选择
根据Ceph官网的描述，"Currently requires Hadoop 1.1.X stable series"，使用Hadoop的release-1.1.2分支，Ceph使用luminous分支

##### 编译Hadoop
```
cd hpath  
git checkout release-1.1.2
```
修改src/mapred/org/apache/hadoop/mapreduce/lib/partition/InputSampler.java第319行
```
K[] samples = (K[])sampler.getSample(inf, job);
```
然后执行
```  
ant compile  
```
`hpath: 替换为相应的hadoop目录的路径`

##### 编译Ceph
```
cd cpath  
git checkout luminous  
./run-make-check.sh
cd build  
make vstart        
../src/vstart.sh -d -n -x -l  
bin/ceph -s  
```
`cpath: 替换为相应的ceph目录的路径`

如果显示如下，则Ceph单机测试集群已经启动  
```
cluster:  
  id:     34d6f7d7-3d0b-46c6-848d-69b0dfe33965  
  health: HEALTH_OK

services:  
  mon: 3 daemons, quorum a,b,c  
  mgr: x(active)  
  mds: cephfs_a-1/1/1 up  {0=c=up:active}, 2 up:standby  
  osd: 3 osds: 3 up, 3 in
```

##### 创建Hadoop使用的pool
```
bin/ceph osd pool create hadoop 100  
bin/ceph osd pool set hadoop size 1  
bin/ceph mds add_data_pool hadoop
```

##### 挂载CephFS
```
mkdir cephfs
sudo mount -t ceph host:port:/ /mnt/cephfs -o name=admin,secret=keyring  
```
`host: ceph mon的ip或者hostname,可以通过ceph mon stat查看`  
`port: ceph mon的端口`  
`keyring: 替换为ceph路径下build/keyring中[client.admin]的key`

##### 安装依赖
添加key
```
wget -q -O- 'https://download.ceph.com/keys/release.asc' | sudo apt-key add -
```
在/etc/apt/sources.list中加入  
```
deb http://mirrors.aliyun.com/ceph/debian-luminous xenial main  
deb-src http://mirrors.aliyun.com/ceph/debian-luminous xenial main  
```
`根据具体的OS和ceph可替换相应的版本`  
```
sudo apt update  
sudo apt install libcephfs-jni libcephfs-java  
ln -s /usr/lib/jni/libcephfs_jni.so hpath/lib/
```

下载Ceph官网提供的插件hadoop-cephfs.jar  
```
cp dpath/hadoop-cephfs.jar hpath/lib/  
```
`hpath: 替换为相应的hadoop目录的路径`  
`dpath: 替换为相应的下载路径`

##### 配置文件
###### hadoop-env.sh
在文件中加入  
```
JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64    
```
`替换为相应的jdk路径`
```
export HADOOP_CLASSPATH=/usr/share/java/libcephfs.jar
```
`通常在/usr/share/java路径下，可以通过locate定位包的位置`

###### core.site.xml
```xml
<configuration>

        <property>
        <name>fs.default.name</name>
        <value>ceph://host:port/</value>
        </property>

        <property>
        <name>ceph.conf.file</name>
        <value>cpath/build/ceph.conf</value>
        </property>

        <property>
        <name>ceph.auth.id</name>
        <value>admin</value>
        </property>

        <property>
        <name>ceph.auth.keyring</name>
        <value>cpath/build/keyring</value>
        </property>

        <property>
        <name>ceph.data.pools</name>
        <value>hadoop</value>
        </property>

        <property>
        <name>fs.ceph.impl</name>
        <value>org.apache.hadoop.fs.ceph.CephFileSystem</value>
        </property>

</configuration>
```
`host: ceph mon的ip或者hostname`  
`port: ceph mon的端口`  
`cpath: 替换为相应的ceph目录的路径`

###### mapred-site.xml
```xml
<configuration>

        <property>
        <name>mapred.job.tracker</name>
        <value>host:9001</value>
        </property>

</configuration>
```
`host: ceph mon的ip`

###### masters
```
        host

```
###### slaves
```
        host
```
`masters和slaves中的host意义同上`

##### 测试
```
cd hpath  
bin/start-mapred.sh  
jps  
```
可以看到JobTracker和TaskTracker即启动成功

##### 上传文件
```
bin/hadoop fs -put README.txt /
```
显示如下

```
Loading libcephfs-jni from default path:  
/usr/java/packages/lib/amd64:/usr/lib/x86_64-linux-gnu/jni:/lib/x86_64-linux-gnu:/usr/lib/x86_64-linux-gnu:/usr/lib/jni:/lib:/usr/lib  
Loading libcephfs-jni: Success!  
```
可以看到上传过程中调用了libcephfs-jni

```
bin/hadoop fs -ls /
```
显示如下

```  
Loading libcephfs-jni from default path:  
/usr/java/packages/lib/amd64:/usr/lib/x86_64-linux-gnu/jni:/lib/x86_64-linux-gnu:/usr/lib/x86_64-linux-gnu:/usr/lib/jni:/lib:/usr/lib  
Loading libcephfs-jni: Success!  
Found 2 items  
drwxrwxrwx   - lab          4 2017-10-25 11:44 /tmp  
-rwxrwxrwx   3 lab       1366 2017-10-25 11:44 /README.txt  
```
