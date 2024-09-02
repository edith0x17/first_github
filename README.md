## 1. Ubuntu 가상환경 설치

1. https://www.virtualbox.org/wiki/Downloads   에서 window hosts로 설치
2. ubuntu iso 설치 https://mirror.kakao.com/ubuntu-releases/  (저는 22.04 받음)
3. Oracle VM virtualbox 관리자 -> 새로만들기 -> iso 이미지 선택 -> 게스트확장 -> 확인
4.  (터미널 안열릴 경우 언어세팅 - English - Canada로 바꿔주고 재시작)



## 2. 기본설치

```bash
sudo apt update
sudo apt install openjdk-8-jdk -y  # java 설치
java -version

sudo apt install ssh -y # ssh 설치
```



## 3. 하둡 설치

```bash
wget https://archive.apache.org/dist/hadoop/common/hadoop-3.3.1/hadoop-3.3.1.tar.gz
tar -xzf hadoop-3.3.1.tar.gz
mv hadoop-3.3.1 hadoop
```



```bash
$ vim ~/.bashrc


export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
export BASE_HOME=/home/[username] #본인의 계정으로 수정 
export HADOOP_HOME=$BASE_HOME/hadoop
export HADOOP_INSTALL=$HADOOP_HOME
export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export YARN_HOME=$HADOOP_HOME
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
# PATH 설정을 통해 해당 경로로 이동하거나 경로를 입력하지 않아도 실행 가능
export PATH=$PATH:$HADOOP_HOME/sbin:$HADOOP_HOME/bin
export HADOOP_OPTS="-Djava.library.path=$HADOOP_HOME/lib/native"

#저장하고 나옴

source ~/.bashrc
```





## 4. Slave 노드 생성

- 가상머신 복제

이때까지는 가상머신 한대에서 master노드를 생성했습니다. 이를 복제하여 slave 노드를 만듭니다.

가상머신 선택 -> 이름수정 -> MAC주소정책 수정 -> 다음 -> 완전한 복제

![image](https://github.com/user-attachments/assets/a9822b82-3cc9-4952-b7e4-a79e9b09b107)



- 호스트 name 변경

```bash
hostnamectl set-hostname slave1 # slave1으로 변경
sudo reboot
```



- IP 확인

![image](https://github.com/user-attachments/assets/5bb2a950-1f26-4daf-bd50-276a91f0b667)



- `/etc/hosts` 수정

  분산 노드들 간의 정보를 알 수 있는 파일을 모든 노드상에서 수정해주어야 합니다.







## 5. SSH 설정 및 암호없이 연결하기

하둡은 노드간에 ssh 통신 프로토콜을 사용해 통신합니다.

```bash
ssh slave1 # 현재 패스워드 입력해야함
```

기존의 ssh [hostname] 을 입력하면 비밀번호를 입력해야 했습니다. 분산 처리를 위한 비밀번호를 생략한 통신을 위해 master노드에서 공개키를 생성해 각각의 slave노드에게 인증된 키로 보내주는 과정이 필요합니다.



```bash
$ ssh-keygen -t rsa -f ~/.ssh/id_rsa
# 엔터

# authorized_keys에 공개키 추가
$ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys

# 권한 변경
$ chmod 0600 ~/.ssh/authorized_keys

# slave로 키 복사
$ ssh-copy-id -i /home/[username]/.ssh/id_rsa.pub slave1
$ ssh-copy-id -i /home/[username]/.ssh/id_rsa.pub slave2
# 계속 연결 할 것인지 물어보면 > yes
# password 입력

$ ssh slave1 # password 없이 연결 확인
$ exit
```



## 6. 하둡 세팅

```bash
cd $HADOOP_HOME/etc/hadoop
```

위 디렉토리로 이동해 진행



**core_site.xml**

```xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://master:9000</value>
    </property>
</configuration>
```



**hdfs-site.xml**

```xml
<configuration>
	<property>
		<name>dfs.data.dir</name>
		<value>/home/[username]/dfsdata/namenode</value> #수정필요
	</property>
	<property>
		<name>dfs.data.dir</name>
		<value>/home/[username]/dfsdata/datanode</value> #수정필요
 	</property>
	<property>
		<name>dfs.replication</name>
		<value>1</value>
	</property>
     <property>
        <name>dfs.namenode.secondary.http-address</name>
        <value>slave1:50090</value>
    </property>
</configuration>
```



**mapred-site.xml**

```xml
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
         <value>yarn</value>
    </property>
</configuration>
```



**yarn-site.xml**

```xml
<configuration>
	<property>
		<name>yarn.nodemanager.aux-services</name>
		<value>mapreduce_shuffle</value>
	</property>
	<property>
		<name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>
		<value>org.apache.hadoop.mapred.ShuffleHandler</value>
	</property>
	<property>
		<name>yarn.resourcemanager.hostname</name>
		<value>localhost</value>
	</property>
	<property>
		<name>yarn.acl.enable</name>
		<value>0</value>
	</property>
	<property>
		<name>yarn.nodemanager.env-whitelist</name>
		<value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS,HADOOP_CONF_DIR,CLASSPATH_PERPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_MAPRED_HOME</value>
	</property>
</configuration>
```



**hadoop-env.sh**

```sh
# / 누르고 export JAVA_HOME 검색, #을 지우고 아래로 수정
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
```



**vim workers**

```
# 모두 지우고
slave1
slave2
...
# 입력 후 저장
```



모든 파일 설정후 모든 slave들에게 복사

```bash
scp $HADOOP_HOME/etc/hadoop/* slave1:$HADOOP_HOME/etc/hadoop/
scp $HADOOP_HOME/etc/hadoop/* slave2:$HADOOP_HOME/etc/hadoop/
...
```



## 7. 하둡 실행

```bash
hdfs namenode -format # 포맷하고 시작 (이거 왜함..?)
start-all.sh # 시작
stop-all.sh # 정지
jps # 상태 확인
```

![image](https://github.com/user-attachments/assets/88175d8f-3a10-413c-9f39-b3b268516887)



jps 명령어 수행시, master노드는 name node를 수행, slave1,2노드는 각각 datanode를 수행하고 있음을 볼 수 있습니다.



이로써 하둡 기본 환경세팅이 끝났습니다.
