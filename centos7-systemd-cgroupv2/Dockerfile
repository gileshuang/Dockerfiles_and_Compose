FROM centos:7
MAINTAINER "huangjiayao <huangjiayao_1992@163.com>"
RUN yum clean all && \
rm -fv /etc/yum.repos.d/CentOS-*.repo && \
curl -s -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
RUN curl -s https://copr.fedorainfracloud.org/coprs/jsynacek/systemd-backports-for-centos-7/repo/epel-7/jsynacek-systemd-backports-for-centos-7-epel-7.repo \
-o /etc/yum.repos.d/jsynacek-systemd-centos-7.repo && \
yum -y install systemd systemd-libs && \
/usr/bin/uuidgen | sed 's/-//g' > /etc/machine-id; \
yum -y update; yum clean all;
VOLUME [ "/sys/fs/cgroup" ]
CMD ["/usr/lib/systemd/systemd"]
