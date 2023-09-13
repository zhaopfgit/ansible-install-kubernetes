# Ansible install kubernetes containerd calico cluster 

查看完整图文文档，请点解下面链接：

使用 ansible 一键安装kubernetes+containerd+calico集群

https://zhpengfei.com/ansible-kubernetes-containerd-calico-install/

因为老电脑用久了，太慢了，近期买了macbook pro m2，安装虚拟机fusion后，想用ansible 快速安装kubernetes集群

下载镜像安装了cenots8-aarch64，之后yum install安装ansible提示有冲突，如下图所属


安装ansible报错
故而放弃yum安装，采用pip安装

## 先安装ansible
```
yum install python39 -y
pip3 install ansible
```
ansible安装成功了

配置免密
```
ssh-copy-id -i /root/.ssh/id_rsa.pub root@172.16.37.130
ssh-copy-id -i /root/.ssh/id_rsa.pub root@172.16.37.131
ssh-copy-id -i /root/.ssh/id_rsa.pub root@172.16.37.132
```
pip安装ansible 没有配置文件，采用默认配置，也可以手动创建
```
ansible-config init --disabled > /etc/ansible/ansible.cfg
mkdir -p /etc/ansible/roles
vim /etc/ansible/hosts
172.16.37.130 master01
172.16.37.131 node01
172.16.37.132 node02
```
最后:wq保存退出
接下来开始编写playbook安装kubernetes机群

由于之前手动安装过kubernetes集群，现将之前手动安装的步骤写入playbook

## Kubernetes 1.25 集群保姆级安装教程，使用 Containerd和Calico网络插件

### 创建kubernetes 模块目录
```
mkdir -pv /etc/ansible/roles/kubernetes{files/tasks/templates/handlers}
tasks:目的: 定义主要执行的任务。它通常包含一个名为 main.yml 的文件，该文件定义了当角色被引用时要执行的任务序列。
files:目的: 存储不需要模板化的静态文件，这些文件可以在任务中直接引用并复制到目标主机，无需任何修改。
templates:目的: 存储 Jinja2 模板文件。模板文件是那些根据变量值生成的文件。例如，你可能有一个配置文件，在不同的主机或环境中需要不同的值。使用模板，你可以动态地生成这个配置文件。
handlers:目的: 定义处理器（handlers）。处理器是根据通知触发的任务，通常用于重启服务或应用配置更改等操作
```
## 编写playbook
```
cd /etc/ansible/roles
vim kubernetes/tasks/main.yaml
- name: 配置hosts
  template: src=hosts.j2 dest=/etc/hosts
- name: SELINUX=disabled
  selinux: state=disabled
- name: firewalld stop
  shell: systemctl stop firewalld;systemctl disable firewalld
- name: swapoff
  shell: swapoff -a;sed -i 's@^/dev/mapper/cl_fedora-swap none                    swap    defaults        0 0@#&@' /etc/fstab
- name:
  shell: modprobe br_netfilter;echo "modprobe br_netfilter" >> /etc/profile
- name:
  copy: src=kubernetes.conf dest=/etc/sysctl.d/kubernetes.conf
- name:
  yum: name=yum-utils state=latest
- name:
  shell: yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
- name:
  shell: yum install -y  device-mapper-persistent-data lvm2 wget net-tools nfs-utils lrzsz gcc gcc-c++ make cmake libxml2-devel openssl-devel curl curl-devel unzip sudo libaio-devel wget vim ncurses-devel autoconf automake zlib-devel epel-release openssh-server socat  ipvsadm conntrack  telnet ipvsadm
- name: 配置kubernetes国内源
  copy: src=kubernetes.repo dest=/etc/yum.repos.d/kubernetes.repo
- name:
  shell: yum clean all;yum makecache
- name: 配置时间同步
  shell: yum install -y chrony
- name: 修改chrony配置
  shell: sed -i 's/#pool 2.centos.pool.ntp.org iburst/server ntp1.aliyun.com iburst\nserver ntp2.aliyun.com iburst\nserver ntp3.aliyun.com iburst/g' /etc/chrony.conf
- name: restart chrony
  service: name=chronyd state=restarted
- name: enable chronyd
  shell: systemctl enable chronyd
- name: 安装containerd服务
  yum: name=containerd.io-1.6.6 state=present
- name:
  shell: containerd config default > /etc/containerd/config.toml
- name:
  shell: sed -i 's#SystemdCgroup = false#SystemdCgroup = true#' /etc/containerd/config.toml
- name:
  shell: sed -i 's#sandbox_image = "k8s.gcr.io/pause:3.6"#sandbox_image="registry.aliyuncs.com/google_containers/pause:3.7"#' /etc/containerd/config.toml
- name: 启动containerd服务
  service: name=containerd state=started enabled=yes
- name:
  copy: src=crictl.yaml    dest=/etc/crictl.yaml
  notify: restart containerd
- name:  安装docker-ce
  yum: name=docker-ce state=present
- name: 目录不存在则创建
  file: path=/etc/docker state=directory
- name: 配置docker镜像加速
  copy: src=daemon.json dest=/etc/docker/daemon.json
  notify: restart docker
- name: 启动docker并开机自动启动
  shell: systemctl enable docker --now
- name:
  shell: sed -i 's#config_path = ""#config_path = "/etc/containerd/certs.d"#' /etc/containerd/config.toml
- name:
  shell: mkdir -p /etc/containerd/certs.d/docker.io/
- name:
  copy: src=hosts.toml dest=/etc/containerd/certs.d/docker.io/hosts.toml
  notify: restart containerd
- name: install kubernetes
  shell: yum install -y kubelet-1.25.0 kubeadm-1.25.0 kubectl-1.25.0
- name:
  shell: systemctl enable kubelet
- name:
  shell: crictl config runtime-endpoint /run/containerd/containerd.sock
- name: tasks for master01
  block:
    - name:
      shell: kubeadm config print init-defaults > kubeadm.yaml
    - name:
      shell: |
        sed -i -e 's#advertiseAddress: 1.2.3.4#advertiseAddress: 172.16.37.130#' -e 's#name: node#name: master01#' -e 's#imageRepository: registry.k8s.io#imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers#' kubeadm.yaml
    - name:
      shell: |
        sed  -i '/  dnsDomain: cluster.local/a\  podSubnet: 10.244.0.0/16' kubeadm.yaml
    - name:
      shell: |
        cat <<EOL >> kubeadm.yaml
        ---
        apiVersion: kubeproxy.config.k8s.io/v1alpha1
        kind: KubeProxyConfiguration
        mode: ipvs
        ---
        apiVersion: kubelet.config.k8s.io/v1beta1
        kind: KubeletConfiguration
        cgroupDriver: systemd
        EOL
    - name: 集群初始化
      shell: kubeadm init --config=kubeadm.yaml --ignore-preflight-errors=SystemVerification
    - name:
      shell: |
        mkdir -p $HOME/.kube;sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config;sudo chown $(id -u):$(id -g) $HOME/.kube/config
  when: "ansible_nodename == 'master01' and ansible_ens160.ipv4.address == '172.16.37.130'"
- name: tasks for node01和node02
  block:
    - name: Get the join command from master01
      command: kubeadm token create --print-join-command
      register: join_command_output
      delegate_to: master01
    - name: join the kubernetes cluster
      command: "{{ join_command_output.stdout }}"
  when: "ansible_nodename in ['node01', 'node02']"
  #- name:
  #  debug: var=join_command_output
  #  when: "ansible_nodename == 'master01' and ansible_ens160.ipv4.address == '172.16.37.130'"
- name:  tasks for master01
  block:
    - name: 下载calico
      shell: wget https://docs.projectcalico.org/manifests/calico.yaml
    - name: 安装calico
      shell: kubectl apply -f  calico.yaml
    - name:  查看系统组建
      shell: kubectl get pod -n kube-system
    - name: 给node节点打标签
      shell: kubectl label nodes node01 node-role.kubernetes.io/work=work;kubectl label nodes node02 node-role.kubernetes.io/work=work
  when: "ansible_nodename == 'master01' and ansible_ens160.ipv4.address == '172.16.37.130'"
```
>部分注解
>>使用block语句

>使用Ansible的block语句来将master01节点上需要执行的步骤放在一起，并在block的结尾附加了一个when条件，且与关键字block对齐

>使用delegate_to功能

>delegate_to是 Ansible 的一个特别有用的参数，它允许我在指定的主机上执行特定的任务，而不是在当前的目标主机上执行，如下面这块，我从上面单独提出来说明

>我想在master上获取加入集群节点的命令，然后在两个node节点上执行，使用 register: join_command_output 来保存 kubeadm token create 的输出，执行上一步输出{{ join_command_output.stdout }}
```
- name: tasks for node01和node02
  block:
    - name: get the join command from master01
      command: kubeadm token create --print-join-command
      register: join_command_output
      delegate_to: master01
    - name: join the kubernetes cluster
      command: "{{ join_command_output.stdout }}"
  when: "ansible_nodename in ['node01', 'node02']"
shell | shell后面竖线，是为了支持多行
<
```
### 使用 handlers
```
vim kubernetes/handlers/main.yml
- name: restart containerd
  service: name=containerd state=restarted
- name: restart docker
  service: name=docker state=restarted
```
### 使用templates
```
vim kubernetes/templates/hosts.j2
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
{{ ansible_ens160.ipv4.address }} {{ ansible_nodename }}
vim kubernetes.yaml
- hosts: all   #选择需要执行的主机组或IP
  remote_user: root  #远程执行用户
  roles:
  - kubernetes
```
## 执行Playbook
```
ansible-playbook kubernetes.yaml
```
##查看kubernetes 集群
```
kubectl get nodes
kubectl get pods -n kube-system
```
集群状态ok,node节点标签也打成功了

