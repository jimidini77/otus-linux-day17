# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
  :selinux => {
        :box_name => "centos/7",
        :box_version => "2004.01",
        :ip_addr => '192.168.11.101',
  },
}

Vagrant.configure("2") do |config|
#config.vm.synced_folder ".", "/vagrant", disabled: true
  MACHINES.each do |boxname, boxconfig|

      config.vm.define boxname do |box|

          box.vm.box = boxconfig[:box_name]
          box.vm.host_name = boxname.to_s

          box.vm.host_name = "selinux"
          box.vm.network "forwarded_port", guest: 4881, host: 4881

          #box.vm.network "private_network", ip: boxconfig[:ip_addr]

          box.vm.provider :virtualbox do |vb|
            	  vb.customize ["modifyvm", :id, "--memory", "1024"]
		  end
 	  box.vm.provision "shell", inline: <<-SHELL
          yum install -y epel-release
          #install nginx
          yum install -y nginx
          yum install -y policycoreutils-python
          #change nginx port
          sed -ie 's/:80/:4881/g' /etc/nginx/nginx.conf
          sed -i 's/listen 80;/listen 4881;/' /etc/nginx/nginx.conf
          #disable SELinux
          #setenforce 0
          #start nginx
          systemctl start nginx
          systemctl status nginx
          #check nginx port
          ss -tlpn | grep 4881
          SHELL
      end
  end
end

