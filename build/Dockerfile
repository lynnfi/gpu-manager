ARG base_img
FROM  10.199.2.148:5000/robocloud/nvidia-cuda:11.0.3-devel-centos7 AS build
#FROM nvidia/cuda:11.0.3-devel-centos7 as build
#FROM nvidia/cuda:11.0.3-devel-ubuntu20.04 as build
#FROM nvidia/cuda:10.1-devel-centos7 as build

ARG version
ARG commit

RUN curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo

RUN sed -i -e '/mirrors.cloud.aliyuncs.com/d' -e '/mirrors.aliyuncs.com/d' /etc/yum.repos.d/CentOS-Base.repo

RUN yum makecache

RUN yum install -y rpm-build make

#RUN yum install -y https://repo.ius.io/ius-release-el7.rpm


#RUN yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm

#RUN yum remove -y git
#RUN yum install -y git
ENV GOLANG_VERSION=1.16.3
RUN curl -sSL https://dl.google.com/go/go${GOLANG_VERSION}.linux-amd64.tar.gz \
    | tar -C /usr/local -xz
ENV GOPATH=/go
ENV PATH=$GOPATH/bin:/usr/local/go/bin:$PATH

RUN export GOPROXY=https://goproxy.cn,direct
RUN export GO111MODULE=on
RUN mkdir -p /root/rpmbuild/{SPECS,SOURCES}

COPY gpu-manager.spec /root/rpmbuild/SPECS
COPY gpu-manager-source.tar.gz /root/rpmbuild/SOURCES

RUN echo '%_topdir /root/rpmbuild' > /root/.rpmmacros \
  && echo '%__os_install_post %{nil}' >> /root/.rpmmacros \
  && echo '%debug_package %{nil}' >> /root/.rpmmacros
WORKDIR /root/rpmbuild/SPECS

RUN go env -w GOPROXY=https://goproxy.cn,direct
RUN rpmbuild -bb --quiet \
  --define 'version '${version}'' \
  --define 'commit '${commit}'' \
  gpu-manager.spec


FROM $base_img

ARG version
ARG commit

COPY --from=build /root/rpmbuild/RPMS/x86_64/gpu-manager-${version}-${commit}.el7.x86_64.rpm /tmp
#RUN mv /root/rpmbuild/RPMS/x86_64/gpu-manager-${version}-${commit}.el7.x86_64.rpm /tmp

RUN curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
RUN sed -i -e '/mirrors.cloud.aliyuncs.com/d' -e '/mirrors.aliyuncs.com/d' /etc/yum.repos.d/CentOS-Base.repo

RUN yum makecache

RUN yum install epel-release -y && \
  yum install -y which jq

# Install packages
RUN rpm -ivh /tmp/gpu-manager-${version}-${commit}.el7.x86_64.rpm \
	&& rm -rf /tmp/gpu-manager-${version}-${commit}.el7.x86_64.rpm

# kubelet
VOLUME ["/var/lib/kubelet/device-plugins"]

# gpu manager storage
VOLUME ["/etc/gpu-manager/vm"]
VOLUME ["/etc/gpu-manager/vdriver"]
VOLUME ["/var/log/gpu-manager"]

# nvidia library search location
VOLUME ["/usr/local/host"]

RUN echo "/usr/local/nvidia/lib" > /etc/ld.so.conf.d/nvidia.conf && \
    echo "/usr/local/nvidia/lib64" >> /etc/ld.so.conf.d/nvidia.conf

ENV PATH=$PATH:/usr/local/nvidia/bin

# cgroup
VOLUME ["/sys/fs/cgroup"]

# display
EXPOSE 5678

COPY start.sh /
COPY copy-bin-lib.sh /

CMD ["/start.sh"]
