# -*- mode: ruby -*-
# vi: set ft=ruby :
servers=[
  {
    :hostname => "minio0",
    :ip => "192.168.200.10"
  },{
   :hostname => "minio1",
   :ip => "192.168.200.11"
  },{
    :hostname => "minio2",
    :ip => "192.168.200.12"  
  }
]

$script = <<SCRIPT
  sudo su root
  yum check-update
  yum makecache fast

  ## Optional: Set timezone
  ## timedatectl set-timezone Asia/Taipei

  ## add docker repo to get correct docker-ce version
  yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

  ## install related packages.
  yum install -y net-tools git wget curl python-pip telnet vim device-mapper-persistent-data lvm2 docker-ce ntp
  
  ## sync system date time
  systemctl start ntpd
  systemctl enable ntpd

  ## install docker-compose
  curl -L https://github.com/docker/compose/releases/download/1.14.0/docker-compose-`uname -s`-`uname -m` > /usr/bin/docker-compose
  chmod +x /usr/bin/docker-compose
  useradd -g docker docker

  ## elasticsearch configuration
  ## https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html#docker-cli-run-prod-mode
  mkdir -p /etc/security/limits.d
  echo "# elasticsearch 5.5.0 configuration" >> /etc/security/limits.d/zz-elastic-limits.conf
  echo "* soft nproc 65535" >> /etc/security/limits.d/zz-elastic-limits.conf
  echo "* hard nproc 65535" >> /etc/security/limits.d/zz-elastic-limits.conf
  echo "* soft nofile 65535" >> /etc/security/limits.d/zz-elastic-limits.conf
  echo "* hard nofile 65535" >> /etc/security/limits.d/zz-elastic-limits.conf
  echo "* soft memlock unlimited" >> /etc/security/limits.d/zz-elastic-limits.conf
  echo "* hard memlock unlimited" >> /etc/security/limits.d/zz-elastic-limits.conf
  mkdir -p /etc/sysctl.d/
  echo "# elasticsearch 5.5.0 configuration" >> /etc/sysctl.d/zz-elastic-sysctl.conf
  echo "vm.max_map_count=262144" >> /etc/sysctl.d/zz-elastic-sysctl.conf
  sysctl -w vm.max_map_count=262144
  ulimit -n 65535
  ulimit -l unlimited

  ## increase docker ulimit
  ## ship default at /usr/lib/systemd/system/docker.service
  ## Drop-In put at /etc/systemd/system/
  mkdir /etc/systemd/system/docker.service.d
  echo "[Service]" >> /etc/systemd/system/docker.service.d/increase-ulimit.conf
  echo "LimitMEMLOCK=infinity" >> /etc/systemd/system/docker.service.d/increase-ulimit.conf

  ## Vagrant data for elasticsearch
  groupadd -f -g 1000 elasticsearch && useradd elasticsearch -ou 1000 -g 1000

  ## data folder
  if [ ! -d "/vagrant/data" ]; then
    mkdir -p /vagrant/data
    chown -R elasticsearch:elasticsearch /vagrant/data
  fi
  
  ## start docker daemon
  systemctl daemon-reload
  systemctl restart docker

  ## Configure service start on boot
  systemctl enable docker

  ## system status
  timedatectl status
  systemctl status docker

  ## join docker swarm
  if [ "$(hostname)" == "minio0" ];then
    docker swarm init --advertise-addr=192.168.200.10
    docker swarm join-token manager -q > /vagrant/token
  fi
  if [ "$(hostname)" == "minio1" ];then
    docker swarm join --token $(cat /vagrant/token) --advertise-addr=192.168.200.11 192.168.200.10:2377
  fi
  if [ "$(hostname)" == "minio2" ];then
    docker swarm join --token $(cat /vagrant/token) --advertise-addr=192.168.200.12 192.168.200.10:2377
  fi
SCRIPT

# This defines the version 2 of vagrant
Vagrant.configure(2) do |config|
  servers.each do |machine|
    config.vm.define machine[:hostname] do |node|
      node.vm.box = "centos/7"
      node.vm.hostname = machine[:hostname]
      node.vm.network "private_network", ip: machine[:ip]
      config.vm.synced_folder ".", "/vagrant", type: "nfs", :linux__nfs_options => ["no_root_squash"], :map_uid => 0, :map_gid => 0
      node.vm.provider "virtualbox" do |vb|
        vb.memory = 2048
        vb.cpus = 1
      end
      node.vm.provision "shell" do |s|
        s.inline = $script
      end
    end
  end
end
