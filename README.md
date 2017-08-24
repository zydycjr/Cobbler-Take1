# Cobbler-Take1

Disable Selinux 加入防火墙

firewall-cmd --permanent --add-port=67/udp
firewall-cmd --permanent --add-port=68/udp
firewall-cmd --permanent --add-port=80/tcp
firewall-cmd --permanent --add-port=443/tcp
firewall-cmd --reload
firewall-cmd --permanent --list-ports

sed -in 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config

cat /etc/selinux/config | egrep -i ^SELINUX

SELINUX=disabled <-- 已經改為 disabled 了
SELINUXTYPE=targeted
reboot  重启后才可以生效

安装与启动所需插件
 yum -y install epel-release
 yum install cobbler cobbler-web dhcp tftp-server pykickstart httpd xinetd fence-agents python-ctypes -y
 systemctl start xinetd.service
 systemctl enable xinetd.service
 systemctl start httpd
 systemctl enable httpd
 systemctl start cobblerd.service
 systemctl enable cobblerd.service
 systemctl start rsyncd.service
 systemctl enable rsyncd.service

下载Bootloader


cobbler get-loaders


配置Cobbler



//检查需要配置的文件
cobbler check 

The following are potential configuration items that you may want to fix:

1 : The 'server' field in /etc/cobbler/settings must be set to something other than localhost, or kickstarting features will not work.  This should be a resolvable hostname or IP for the boot server as reachable by all machines that will use it.

2 : For PXE to be functional, the 'next_server' field in /etc/cobbler/settings must be set to something other than 127.0.0.1, and should match the IP of the boot server on the PXE network.

3 : change 'disable' to 'no' in /etc/xinetd.d/tftp

4 : some network boot-loaders are missing from /var/lib/cobbler/loaders, you may run 'cobbler get-loaders' to download them, or, if you only want to handle x86/x86_64 netbooting, you may ensure that you have installed a *recent* version of the syslinux package installed and can ignore this message entirely.  Files in this directory, should you want to support all architectures, should include pxelinux.0, menu.c32, elilo.efi, and yaboot. The 'cobbler get-loaders' command is the easiest way to resolve these requirements.

5 : enable and start rsyncd.service with systemctl

6 : debmirror package is not installed, it will be required to manage debian deployments and repositories

7 : The default password used by the sample templates for newly installed machines (default_password_crypted in /etc/cobbler/settings) is still set to 'cobbler' and should be changed, try: "openssl passwd -1 -salt 'random-phrase-here' 'your-password-here'" to generate new one

: fencing tools were not found, and are required to use the (optional) power management features. install cman or fence-agents to use them
Restart cobblerd and then run 'cobbler sync' to apply changes.

//修改全局设定
nano /etc/cobbler/settings 
     # if you do not set this correctly, this will be manifested in TFTP open timeouts.
     将“next_server: 127.0.0.1”修改为“next_server: 192.168.x.x”
     将“server: 127.0.0.1”修改为“server: 192.168.x.x”
     # set to 1 to enable Cobbler's DHCP management features.
     # the choice of DHCP management engine is in /etc/cobbler/modules.conf
     将“manage_dhcp: 0”修改为“manage_dhcp: 1”
 
//或者 后面ip为服务器所在ip
sed -i "s/server: 127.0.0.1/server: 192.168.x.x/g" /etc/cobbler/settings
sed -i "s/next_server: 127.0.0.1/next_server: 192.168.x.x/g" /etc/cobbler/settings
sed -i 's/manage_dhcp: 0/manage_dhcp: 1/' /etc/cobbler/settings
sed -i 's/pxe_just_once: 0/pxe_just_once: 1/' /etc/cobbler/settings
cat /etc/cobbler/settings | egrep "^next_server|^server"    //检查一下

//TFTP 服務是由 xinetd 這個 daemon 來管理的，因此我們去修改 /etc/xinetd.d/tftp 這個設定檔。
sed -i '14s/yes/no/' /etc/xinetd.d/tftp
cat /etc/xinetd.d/tftp


//修改系统安装后root密码
nano  /etc/cobbler/settings
//第一个值加点盐随便值 第二为密码
openssl passwd -1 -salt 'kaybaba' '111111'
//得到的值覆盖原值
default_password_crypted:  xxxxxxxxx

//配置DHCP
nano /etc/cobbler/dhcp.template

subnet 192.168.141.0 netmask 255.255.255.0 {
     option routers             192.168.141.1;                                  /*网关地址*/
     option domain-name-servers 168.95.1.1;                              /*DNS*/
     option subnet-mask         255.255.255.0;                              /*子网掩码*/
     range dynamic-bootp        192.168.141.50 192.168.141.100;    /*ip地址段范围*/
     default-lease-time         21600;
     max-lease-time             43200;
     next-server                $next_server;
     class "pxeclients" {
          match if substring (option vendor-class-identifier, 0, 9) = "PXEClient";
          if option pxe-system-type = 00:02 {
                  filename "ia64/elilo.efi";
          } else if option pxe-system-type = 00:06 {
                  filename "grub/grub-x86.efi";
          } else if option pxe-system-type = 00:07 {
                  filename "grub/grub-x86_64.efi";
          } else {
                  filename "pxelinux.0";
          }
     }

}



都完成后
systemctl restart cobblerd.service
cobbler sync







可选步骤:

	* 配置cobber_web

useradd admin
passwd admin
sed -i 's/admin = ""/admin = "admin"/' /etc/cobbler/users.conf
sed -i 's/module = authn_configfile/module = authn_pam/' /etc/cobbler/modules.conf 


Cobbler使用


cobbler –name:倒进后的名字 –arch:镜像架构 –path:挂载路径

mount /iso /mnt/
cobbler import --path=/mnt/ --name=CentOS --arch=x86_64

导入后镜像所在位置：/var/www/cobbler/ks_mirror/


cobbler profile edit --name=CentOS-7.3.1611-x86_64 --kickstart=/var/lib/cobbler/kickstarts/CentOS-7.3.1611-x86_64.ks
