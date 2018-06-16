# 基础步骤

- 环境清单及/etc/hosts文件，操作系统为7.4

```
172.16.37.41 local2-ocp3.9-master1-test.com 
172.16.37.42 local2-ocp3.9-node1-test.com 
172.16.37.43 local2-ocp3.9-node2-test.com 
172.16.37.43 local2-ocp3.9-yum-test.com 
172.16.37.43 local2-ocp3.9-registry-test.com 
172.16.37.41 local2-ocp3.9-nfs-test.com 
```

- 配置主机名及hosts文件

```
hostnamectl set-hostname local2-ocp3.9-node2-test.com
```

- 配置selinux为enforcing

```
[root@local2-ocp3.9-node2-test.com ~]# cat /etc/selinux/config

# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=enforcing
# SELINUXTYPE= can take one of three two values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
```

# yum 源搭建(172.16.37.43)

- 上传openshift RPM包至节点，并搭建为yum源

```
[root@local2-ocp3.9-node2-test.com ~]# ll /opt/repos/
总用量 0
drwxr-xr-x. 4 root root 55 4月  28 12:12 rhel-7-fast-datapath-rpms
drwxr-xr-x. 4 root root 55 4月  28 12:12 rhel-7-server-ansible-2.4-rpms
drwxr-xr-x. 4 root root 55 4月  28 12:12 rhel-7-server-extras-rpms
drwxr-xr-x. 4 root root 55 4月  28 12:25 rhel-7-server-ose-3.9-rpms
drwxr-xr-x. 5 root root 86 4月  28 12:12 rhel-7-server-rpms
```

- 创建本机yum源

```
[root@local2-ocp3.9-node2-test.com ~]# cat /etc/yum.repos.d/ocp-local.repo
[rhel-7-server-rpms]
name=rhel-7-server-rpms
baseurl=file:///opt/repos/rhel-7-server-rpms
enabled=1
gpgcheck=0
[rhel-7-server-extras-rpms]
name=rhel-7-server-extras-rpms
baseurl=file:///opt/repos/rhel-7-server-extras-rpms
enabled=1
gpgcheck=0
[rhel-7-fast-datapath-rpms]
name=rhel-7-fast-datapath-rpms
baseurl=file:///opt/repos/rhel-7-fast-datapath-rpms
enabled=1
gpgcheck=0
[rhel-7-server-ose-3.9-rpms]
name=rhel-7-server-ose-3.9-rpms
baseurl=file:///opt/repos/rhel-7-server-ose-3.9-rpms
enabled=1
gpgcheck=0
[rhel-7-server-ansible-2.4-rpms]
name=rhel-7-server-ansible-2.4-rpms
baseurl=file:///opt/repos/rhel-7-server-ansible-2.4-rpms
enabled=1
gpgcheck=0
```

- 查看yum源是否可用

```
[root@local2-ocp3.9-node2-test.com ~]# yum repolist
已加载插件：product-id, search-disabled-repos, subscription-manager
This system is not registered with an entitlement server. You can use subscription-manager to register.
rhel-7-fast-datapath-rpms                                                                                                                             | 2.9 kB  00:00:00
rhel-7-server-ansible-2.4-rpms                                                                                                                        | 2.9 kB  00:00:00
rhel-7-server-extras-rpms                                                                                                                             | 2.9 kB  00:00:00
rhel-7-server-ose-3.9-rpms                                                                                                                            | 2.9 kB  00:00:00
rhel-7-server-rpms                                                                                                                                    | 2.9 kB  00:00:00
(1/5): rhel-7-fast-datapath-rpms/primary_db                                                                                                           |  24 kB  00:00:00
(2/5): rhel-7-server-ansible-2.4-rpms/primary_db                                                                                                      | 8.7 kB  00:00:00
(3/5): rhel-7-server-extras-rpms/primary_db                                                                                                           |  97 kB  00:00:00
(4/5): rhel-7-server-ose-3.9-rpms/primary_db                                                                                                          | 310 kB  00:00:00
(5/5): rhel-7-server-rpms/primary_db                                                                                                                  | 7.4 MB  00:00:00
源标识                                                                              源名称                                                                              状态
rhel-7-fast-datapath-rpms                                                           rhel-7-fast-datapath-rpms                                                              32
rhel-7-server-ansible-2.4-rpms                                                      rhel-7-server-ansible-2.4-rpms                                                         13
rhel-7-server-extras-rpms                                                           rhel-7-server-extras-rpms                                                             206
rhel-7-server-ose-3.9-rpms                                                          rhel-7-server-ose-3.9-rpms                                                            543
rhel-7-server-rpms                                                                  rhel-7-server-rpms                                                                  7,431
repolist: 8,225
```

- 将本机源设置为yum服务器，安装httpd

```
yum -y install httpd
```

- 配置httpd：/etc/httpd/conf.d/yum.conf

```
Alias /repos "/opt/repos"
<Directory "/opt/repos">
  Options +Indexes +FollowSymLinks
Require all granted
</Directory>
<Location /repos>
SetHandler None
</Location>
```

- 启动httpd

```
systemctl enable httpd;systemctl restart httpd
```

- 检查httpd是否配置正确

```
[root@local2-ocp3.9-node2-test.com ~]# curl local2-ocp3.9-node2-test.com/repos/
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<html>
 <head>
  <title>Index of /repos</title>
 </head>
 <body>
<h1>Index of /repos</h1>
  <table>
   <tr><th valign="top"><img src="/icons/blank.gif" alt="[ICO]"></th><th><a href="?C=N;O=D">Name</a></th><th><a href="?C=M;O=A">Last modified</a></th><th><a href="?C=S;O=A">Size</a></th><th><a href="?C=D;O=A">Description</a></th></tr>
   <tr><th colspan="5"><hr></th></tr>
<tr><td valign="top"><img src="/icons/back.gif" alt="[PARENTDIR]"></td><td><a href="/">Parent Directory</a>       </td><td>&nbsp;</td><td align="right">  - </td><td>&nbsp;</td></tr>
<tr><td valign="top"><img src="/icons/folder.gif" alt="[DIR]"></td><td><a href="rhel-7-fast-datapath-rpms/">rhel-7-fast-datapath..&gt;</a></td><td align="right">2018-04-28 12:12  </td><td align="right">  - </td><td>&nbsp;</td></tr>
<tr><td valign="top"><img src="/icons/folder.gif" alt="[DIR]"></td><td><a href="rhel-7-server-ansible-2.4-rpms/">rhel-7-server-ansibl..&gt;</a></td><td align="right">2018-04-28 12:12  </td><td align="right">  - </td><td>&nbsp;</td></tr>
<tr><td valign="top"><img src="/icons/folder.gif" alt="[DIR]"></td><td><a href="rhel-7-server-extras-rpms/">rhel-7-server-extras..&gt;</a></td><td align="right">2018-04-28 12:12  </td><td align="right">  - </td><td>&nbsp;</td></tr>
<tr><td valign="top"><img src="/icons/folder.gif" alt="[DIR]"></td><td><a href="rhel-7-server-ose-3.9-rpms/">rhel-7-server-ose-3...&gt;</a></td><td align="right">2018-04-28 12:25  </td><td align="right">  - </td><td>&nbsp;</td></tr>
<tr><td valign="top"><img src="/icons/folder.gif" alt="[DIR]"></td><td><a href="rhel-7-server-rpms/">rhel-7-server-rpms/</a>    </td><td align="right">2018-04-28 12:12  </td><td align="right">  - </td><td>&nbsp;</td></tr>
   <tr><th colspan="5"><hr></th></tr>
</table>
</body></html>
```

- 检查yum源中openshift rpm内容是否正确

```
[root@local2-ocp3.9-node2-test.com ~]# yum list|grep atomic-openshift
atomic-openshift.x86_64         3.9.25-1.git.0.6bc473e.el7
atomic-openshift-clients.x86_64 3.9.25-1.git.0.6bc473e.el7
atomic-openshift-clients-redistributable.x86_64
atomic-openshift-cluster-capacity.x86_64
atomic-openshift-docker-excluder.noarch
atomic-openshift-dockerregistry.x86_64
atomic-openshift-excluder.noarch
atomic-openshift-federation-services.x86_64
atomic-openshift-master.x86_64  3.9.25-1.git.0.6bc473e.el7
atomic-openshift-node.x86_64    3.9.25-1.git.0.6bc473e.el7
atomic-openshift-pod.x86_64     3.9.25-1.git.0.6bc473e.el7
atomic-openshift-sdn-ovs.x86_64 3.9.25-1.git.0.6bc473e.el7
atomic-openshift-service-catalog.x86_64
atomic-openshift-template-service-broker.x86_64
atomic-openshift-tests.x86_64   3.9.25-1.git.0.6bc473e.el7
atomic-openshift-utils.noarch   3.9.14-1.git.3.c62bc34.el7
atomic-openshift-web-console.x86_64
```

- 配置yum文件ocp.repo，将原先的ocp-local.repo修改为ocp-local.repo.bak1

```
[root@local2-ocp3.9-node2-test.com yum.repos.d]# cat /etc/yum.repos.d/ocp.repo
[rhel-7-server-rpms]
name=rhel-7-server-rpms
baseurl=http://local2-ocp3.9-yum-test.com/repos/rhel-7-server-rpms
enabled=1
gpgcheck=0
[rhel-7-server-extras-rpms]
name=rhel-7-server-extras-rpms
baseurl=http://local2-ocp3.9-yum-test.com/repos/rhel-7-server-extras-rpms
enabled=1
gpgcheck=0
[rhel-7-fast-datapath-rpms]
name=rhel-7-fast-datapath-rpms
baseurl=http://local2-ocp3.9-yum-test.com/repos/rhel-7-fast-datapath-rpms
enabled=1
gpgcheck=0
[rhel-7-server-ose-3.9-rpms]
name=rhel-7-server-ose-3.9-rpms
baseurl=http://local2-ocp3.9-yum-test.com/repos/rhel-7-server-ose-3.9-rpms
enabled=1
gpgcheck=0
[rhel-7-server-ansible-2.4-rpms]
name=rhel-7-server-ansible-2.4-rpms
baseurl=http://local2-ocp3.9-yum-test.com/repos/rhel-7-server-ansible-2.4-rpms
enabled=1
gpgcheck=0
```

- 检查yum源服务器是否配置完成

```
[root@local2-ocp3.9-node2-test.com yum.repos.d]# yum clean all;yum repolist
已加载插件：product-id, search-disabled-repos, subscription-manager
This system is not registered with an entitlement server. You can use subscription-manager to register.
正在清理软件源： rhel-7-fast-datapath-rpms rhel-7-server-ansible-2.4-rpms rhel-7-server-extras-rpms rhel-7-server-ose-3.9-rpms rhel-7-server-rpms
Cleaning up everything
Maybe you want: rm -rf /var/cache/yum, to also free up space taken by orphaned data from disabled or removed repos
已加载插件：product-id, search-disabled-repos, subscription-manager
This system is not registered with an entitlement server. You can use subscription-manager to register.
rhel-7-fast-datapath-rpms                                                                                                                             | 2.9 kB  00:00:00
rhel-7-server-ansible-2.4-rpms                                                                                                                        | 2.9 kB  00:00:00
rhel-7-server-extras-rpms                                                                                                                             | 2.9 kB  00:00:00
rhel-7-server-ose-3.9-rpms                                                                                                                            | 2.9 kB  00:00:00
rhel-7-server-rpms                                                                                                                                    | 2.9 kB  00:00:00
(1/5): rhel-7-fast-datapath-rpms/primary_db                                                                                                           |  24 kB  00:00:00
(2/5): rhel-7-server-ansible-2.4-rpms/primary_db                                                                                                      | 8.7 kB  00:00:00
(3/5): rhel-7-server-extras-rpms/primary_db                                                                                                           |  97 kB  00:00:00
(4/5): rhel-7-server-ose-3.9-rpms/primary_db                                                                                                          | 310 kB  00:00:00
(5/5): rhel-7-server-rpms/primary_db                                                                                                                  | 7.4 MB  00:00:00
源标识                                                                              源名称                                                                              状态
rhel-7-fast-datapath-rpms                                                           rhel-7-fast-datapath-rpms                                                              32
rhel-7-server-ansible-2.4-rpms                                                      rhel-7-server-ansible-2.4-rpms                                                         13
rhel-7-server-extras-rpms                                                           rhel-7-server-extras-rpms                                                             206
rhel-7-server-ose-3.9-rpms                                                          rhel-7-server-ose-3.9-rpms                                                            543
rhel-7-server-rpms                                                                  rhel-7-server-rpms                                                                  7,431
repolist: 8,225
```

- 创建免密登录

```
[root@local2-ocp3.9-node2-test.com ~]# ssh-keygen
[root@local2-ocp3.9-node2-test.com ~]# ssh-copy-id local2-ocp3.9-node1-test.com
[root@local2-ocp3.9-node2-test.com ~]# ssh-copy-id local2-ocp3.9-node2-test.com
[root@local2-ocp3.9-node2-test.com ~]# ssh-copy-id local2-ocp3.9-master1-test.com
[root@local2-ocp3.9-node2-test.com ~]# ssh-copy-id local2-ocp3.9-registry-test.com
[root@local2-ocp3.9-node2-test.com ~]# ssh-copy-id local2-ocp3.9-yum-test.com
[root@local2-ocp3.9-node2-test.com ~]# ssh-copy-id local2-ocp3.9-nfs-test.com
```

- 将/etc/yum.repos.d/ocp.repo文件在其他节点配置

```
for i in  local2-ocp3.9-node1-test.com local2-ocp3.9-master1-test.com 
do
echo $i
scp /etc/yum.repos.d/ocp.repo $i:/etc/yum.repos.d/ocp.repo
done;
```

- 验证其他节点yum源是否正常

```
for i in  local2-ocp3.9-node1-test.com local2-ocp3.9-master1-test.com 
do
echo $i
ssh $i 'yum repolist'
done;
```

# 所有节点基础包安装

- 基础包

```
yum install wget git net-tools bind-utils yum-utils iptables-services bridge-utils bash-completion kexec-tools sos psacct vim lrzsz python-setuptools unzip tree atomic-openshift-utils docker-1.13.1 -y
```

- 更新

```
yum -y update
```

- 配置iptables，放行流量

```
cp /etc/sysconfig/iptables /etc/sysconfig/iptables.bak.$(date "+%Y%m%d%H%M%S");
sed -i '/.*--dport 22 -j ACCEPT.*/a\-A INPUT -p tcp -m state --state NEW -m tcp --dport 53 -j ACCEPT' /etc/sysconfig/iptables;
sed -i '/.*--dport 22 -j ACCEPT.*/a\-A INPUT -p udp -m state --state NEW -m udp --dport 53 -j ACCEPT' /etc/sysconfig/iptables;
sed -i '/.*--dport 22 -j ACCEPT.*/a\-A INPUT -p tcp -m state --state NEW -m tcp --dport 5000 -j ACCEPT' /etc/sysconfig/iptables;
sed -i '/.*--dport 22 -j ACCEPT.*/a\-A INPUT -p tcp -m state --state NEW -m tcp --dport 80 -j ACCEPT' /etc/sysconfig/iptables;

systemctl restart iptables;
```

- 启动docker服务

```
systemctl start docker;systemctl enable docker
```

# 镜像仓库搭建(172.16.37.43)

- 安装docker-distribution

```
yum install -y docker-distribution;systemctl start docker-distribution;systemctl enable docker-distribution
```

- 配置/etc/containers/registries.conf为如下内容

```
# This is a system-wide configuration file used to
# keep track of registries for various container backends.
# It adheres to TOML format and does not support recursive
# lists of registries.

# The default location for this configuration file is /etc/containers/registries.conf.

# The only valid categories are: 'registries.search', 'registries.insecure',
# and 'registries.block'.

[registries.search]
registries = ['local2-ocp3.9-registry-test.com:5000','172.30.0.0/16']

# If you need to access insecure registries, add the registry's fully-qualified name.
# An insecure registry is one that does not have a valid SSL certificate or only does HTTP.
[registries.insecure]
registries = ['local2-ocp3.9-registry-test.com:5000','172.30.0.0/16']


# If you need to block pull access from a registry, uncomment the section below
# and add the registries fully-qualified name.
#
# Docker only
[registries.block]
registries = ['registry.access.redhat.com']
```

- 启动docker服务

```
systemctl start docker;systemctl enable docker
```

- 上传镜像包并导入，此过程会占用大量磁盘空间和内存资源

```
docker load -i ocp39image.tar.bz2
```

- 修改镜像tag

```
docker images|grep access.redhat|awk '{print $1"/"$2}'|awk -F "/" '{print "docker tag "$1"/"$2"/"$3":"$4" local2-ocp3.9-registry-test.com:5000/"$2"/"$3":"$4}'|sh
```

- 需要额外修改的tag

```
[root@local2-ocp3.9-node2-test.com ~]# docker tag local2-ocp3.9-registry-test.com:5000/openshift3/ose-haproxy-router:v3.9.25-1 local2-ocp3.9-registry-test.com:5000/openshift3/ose-haproxy-router:v3.9.25
[root@local2-ocp3.9-node2-test.com ~]# docker tag local2-ocp3.9-registry-test.com:5000/openshift3/ose-docker-registry:v3.9.25-1 local2-ocp3.9-registry-test.com:5000/openshift3/ose-docker-registry:v3.9.25
```

- 上传镜像至镜像仓库

```
docker images|grep local2-ocp3.9-registry-test.com:5000|awk '{print "docker push " $1":"$2}'|sh
```

# openshift部署节点组件(172.16.37.43)

- 配置ansible hosts文件:/etc/ansible/hosts

```
[OSEv3:children]
masters
nodes
etcd

[OSEv3:vars]
ansible_ssh_user=root
openshift_deployment_type=openshift-enterprise
openshift_image_tag=v3.9.25-1

openshift_disable_check=memory_availability,disk_availability

os_sdn_network_plugin_name=redhat/openshift-ovs-multitenant

oreg_url=local2-ocp3.9-registry-test.com:5000/openshift3/ose-${component}:${version}
openshift_examples_modify_imagestreams=true
openshift_docker_additional_registries=local2-ocp3.9-registry-test.com:5000,172.30.0.1/16:5000
openshift_docker_insecure_registries=local2-ocp3.9-registry-test.com:5000,172.30.0.1/16:5000

openshift_disable_check=memory_availability,disk_availability,package_version
openshift_master_default_subdomain=apps.test.com
openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true', 'kind': 'HTPasswdPasswordIdentityProvider', 'filename': '/etc/origin/master/htpasswd'}]
openshift_master_cluster_method=native
openshift_master_cluster_hostname=cluster.test.com
openshift_master_cluster_public_hostname=cluster.test.com

openshift_enable_service_catalog=false

openshift_metrics_install_metrics=true
openshift_hosted_metrics_public_url=https://hawkular-metrics.apps.test.com/hawkular/metrics
openshift_metrics_image_prefix=local2-ocp3.9-registry-test.com:5000/openshift3/
openshift_metrics_image_version=v3.9.25-1

openshift_logging_install_logging=true
openshift_logging_image_prefix=local2-ocp3.9-registry-test.com:5000/openshift3/
openshift_logging_image_version=v3.9.25-1

openshift_router_selector='router=true'
openshift_registry_selector='router=true'


openshift_clock_enabled=true

openshift_web_console_prefix=local2-ocp3.9-registry-test.com:5000/openshift3/ose-
openshift_web_console_version=v3.9.25

openshift_node_kubelet_args={'pods-per-core': ['10'], 'max-pods':['60']}

[masters]
local2-ocp3.9-master1-test.com
[etcd]
local2-ocp3.9-master1-test.com
[nodes]
local2-ocp3.9-master1-test.com
local2-ocp3.9-node1-test.com openshift_node_labels="{'region': 'infra', 'zone': 'default','infra': 'true','router': 'true'}"
local2-ocp3.9-node2-test.com openshift_node_labels="{'region': 'infra', 'zone': 'default','infra': 'true','router': 'true'}"
```

- 开始部署

```
ansible-playbook /usr/share/ansible/openshift-ansible/playbooks/deploy_cluster.yml
```

- 卸载

```
ansible-playbook  /usr/share/ansible/openshift-ansible/playbooks/adhoc/uninstall.yml
```

