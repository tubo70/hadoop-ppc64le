# 基于Almalinux 8.9构建PPC64le架构的Hadoop Docker镜像

因为Hadoop官方的Dockerfile没有考虑支持PPC64le，所以基于官方的Dockerfile做了简单的修改。

官方地址：https://github.com/apache/hadoop/tree/docker-hadoop-runner-jdk11

这里把修改的地方一并保存下来好对照参考，本次修改主要为了在Power8 S822L的服务器上能够通过K8S部署hadoop：

``` yaml
# FROM centos
# centos 8 不提供支持了，而almalinux 9或rockylinux 9需要CPU ILA支持3.0，这需要Power9才支持
# 而rockylinux 8没有出支持PowerCPU的版本，所以这里选择almlinux 8.9
FROM almalinux:8.9
# RUN rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
RUN yum install -y sudo python2-pip wget nmap-ncat jq java-11-openjdk
RUN pip install robotframework
# RUN wget -O /usr/local/bin/dumb-init https://github.com/Yelp/dumb-init/releases/download/v1.2.0/dumb-init_1.2.0_amd64
# dumb-init 更换成支持ppc64le的版本，Dockerfile的主要修改就是这里
RUN wget -O /usr/local/bin/dumb-init https://github.com/Yelp/dumb-init/releases/download/v1.2.5/dumb-init_1.2.5_ppc64le
RUN chmod +x /usr/local/bin/dumb-init
RUN mkdir -p /etc/security/keytabs && chmod -R a+wr /etc/security/keytabs 
ADD https://repo.maven.apache.org/maven2/org/jboss/byteman/byteman/4.0.4/byteman-4.0.4.jar /opt/byteman.jar
RUN chmod o+r /opt/byteman.jar
# 因为没有找到async-profiler的ppc64le的版本，所以不安装async-profiler，应该不会有什么影响
# RUN mkdir -p /opt/profiler && \
#    cd /opt/profiler && \
#    curl -L https://github.com/jvm-profiling-tools/async-profiler/releases/download/v1.5/async-profiler-1.5-linux-x64.tar.gz | tar xvz
ENV JAVA_HOME=/usr/lib/jvm/jre/
ENV PATH $PATH:/opt/hadoop/bin

RUN groupadd --gid 1000 hadoop
RUN useradd --uid 1000 hadoop --gid 100 --home /opt/hadoop
RUN chmod 755 /opt/hadoop
RUN echo "hadoop ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
RUN chown hadoop /opt
ADD scripts /opt/
ADD scripts/krb5.conf /etc/
RUN yum install -y krb5-workstation
RUN mkdir -p /etc/hadoop && mkdir -p /var/log/hadoop && chmod 1777 /etc/hadoop && chmod 1777 /var/log/hadoop
ENV HADOOP_LOG_DIR=/var/log/hadoop
ENV HADOOP_CONF_DIR=/etc/hadoop
WORKDIR /opt/hadoop
RUN mkdir /data && chmod 1777 /data
USER hadoop
ENTRYPOINT ["/usr/local/bin/dumb-init", "--", "/opt/starter.sh"]
# 以下内容来自：https://github.com/apache/hadoop/tree/docker-hadoop-3
# 官方的Dockerfile分成两个，以上是一个Base Image的Dockerfile
# 下面部分则是安装具体的Hadoop版本

# ARG HADOOP_URL=https://dlcdn.apache.org/hadoop/common/hadoop-3.3.6/hadoop-3.3.6.tar.gz
# 修改成3.4.0版本
ARG HADOOP_URL=https://dlcdn.apache.org/hadoop/common/hadoop-3.4.0/hadoop-3.4.0.tar.gz
WORKDIR /opt
RUN sudo rm -rf /opt/hadoop && curl -LSs -o hadoop.tar.gz $HADOOP_URL && tar zxf hadoop.tar.gz && rm hadoop.tar.gz && mv hadoop* hadoop && rm -rf /opt/hadoop/share/doc
WORKDIR /opt/hadoop
ADD log4j.properties /opt/hadoop/etc/hadoop/log4j.properties
RUN sudo chown -R hadoop:users /opt/hadoop/etc/hadoop/*
ENV HADOOP_CONF_DIR /opt/hadoop/etc/hadoop
```

开始构建，这个目录结构如下：

![image-20240425164510673](assets\image-20240425164510673.png)

``` bash
# k8s集群上安装的是containerd，所以使用nerdctl来构建
# 之前用docker的registry搭建了一个docker的私有仓库，所以构建的时候直接使用私有仓库的标签
nerdctl build -t 127.0.0.1:5000/apache/hadoop:ppc64le .

# 构建完成后发布到私有仓库
nerdctl push 127.0.0.1:5000/apache/hadoop:ppc64le
```



