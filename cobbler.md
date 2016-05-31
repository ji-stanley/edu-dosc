#cobbler安装教程

##安装cobbler
修改yum源进行安装cobbler和依赖服务
<pre>
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
yum -y install cobbler cobbler-web dhcp tftp-server pykickstart httpd xinetd
</pre>
##启动cobbler
<pre>
systemctl restart httpd.service
systemctl restart cobblerd.service
</pre>
##cobbler check
<pre>
cobbler check

The following are potential configuration items that you may want to fix:
1 : The 'server' field in /etc/cobbler/settings must be set to something other than localhost, or kickstarting features will not work.  This should be a resolvable hostname or IP for the boot server as reachable by all machines that will use it.
2 : For PXE to be functional, the 'next_server' field in /etc/cobbler/settings must be set to something other than 127.0.0.1, and should match the IP of the boot server on the PXE network.
3 : change 'disable' to 'no' in /etc/xinetd.d/rsync
4 : file /etc/xinetd.d/rsync does not exist
5 : since iptables may be running, ensure 69, 80/443, and 25151 are unblocked
6 : debmirror package is not installed, it will be required to manage debian deployments and repositories
7 : The default password used by the sample templates for newly installed machines (default_password_crypted in /etc/cobbler/settings) is still set to 'cobbler' and should be changed, try: "openssl passwd -1 -salt 'random-phrase-here' 'your-password-here'" to generate new one
8 : fencing tools were not found, and are required to use the (optional) power management features. install cman or fence-agents to use them
Restart cobblerd and then run 'cobbler sync' to apply changes.
</pre>
##修改cobbler
* 修改cobbler服务IP地址
<pre>
cp /etc/cobbler/settings{,.ori}
sed -i 's/server: 127.0.0.1/server: 192.168.56.11/' /etc/cobbler/settings 
sed -i 's/next_server: 127.0.0.1/next_server: 192.168.56.11/' /etc/cobbler/settings  
sed -i 's/manage_dhcp: 0/manage_dhcp: 1/' /etc/cobbler/settings
sed -i 's/pxe_just_once: 0/pxe_just_once: 1/' /etc/cobbler/settings
</pre>
* 生成加密密码，放入cobbler配置文件
<pre>
vim /etc/cobbler/settings 
default_password_crypted: "$1$jifucha$KskGtEMcVfpvBSaZXI2YQ."
</pre>
* 生成方法
<pre>
openssl passwd -1 -salt 'jifucha' '123456'      
$1$jifucha$KskGtEMcVfpvBSaZXI2YQ.
</pre>
* 下载官方提供的一些包
<pre>
cobbler get-loaders
</pre>
* 修改tftp
<pre>
vim /etc/xinetd.d/tftp 
disable                 = no 
#yes改为no
</pre>
* 重启cobbler
<pre>
systemctl restart cobblerd
cobbler check 
</pre>
* 修改dhcp
<pre>
vim /etc/cobbler/dhcp.template
subnet 192.168.56.0 netmask 255.255.255.0 {
     option routers             192.168.56.2;
     option domain-name-servers 192.168.56.2;
     option subnet-mask         255.255.255.0;
     range dynamic-bootp        192.168.56.100 192.168.56.200;
</pre>
* 挂载复制镜像文件
<pre>
mount /dev/cdrom /mnt/  
cobbler import --path=/mnt/ --name=CentOS-7-x86_64 --arch=x86_64
</pre>
* 查看镜像文件
<pre>
cobbler distro list
cd /var/www/cobbler/ks_mirror/
ls CentOS-7-x86_64/ 
</pre>
* 增加yum源
<pre>
cobbler repo add --name=openstack-mitaka --mirror=http://mirrors.aliyun.com/centos/7.2.1511/cloud/x86_64/openstack-mitaka/ --arch=x86_64 --breed=yum
cobbler reposync 
</pre>
* 修改ks配置文件
```	
cd /var/lib/cobbler/kickstarts/
vim  CentOS-7-x86_64.cfg
----------------------------------------------------------------------------------
# Cobbler for Kickstart Configurator for CentOS 7 by jifucha 
install
url --url=$tree 
text
lang en_US.UTF-8
keyboard us
zerombr
bootloader --location=mbr --driveorder=sda --append="crashkernel=auto rhgb quiet"
# Network information
$SNIPPET('network_config')
timezone --utc Asia/Shanghai
authconfig --enableshadow --passalgo=sha512
rootpw  --iscrypted $default_password_crypted
clearpart --all --initlabel
part /boot --fstype xfs --size 1024  
part swap --size 1024
part / --fstype xfs --size 1 --grow
firstboot --disable
selinux --disabled
firewall --disabled
logging --level=info
reboot
 
%pre
$SNIPPET('log_ks_pre')
$SNIPPET('kickstart_start')
$SNIPPET('pre_install_network_config')
# Enable installation monitoring
$SNIPPET('pre_anamon')
%end
 
%packages
@base
@compat-libraries
@debugging
tree
nmap
lrzsz
dos2unix
openssl-devel
zlib-devel
screen
%end

%post
systemctl disable postfix.service
$yum_config_stanza
%end
---------------------------------------------------------------------------------
cobbler list
```

* 查看镜像
<pre>
cobbler distro report --name=CentOS-7-x86_64
cobbler profile report
</pre>
* 指定ks文件、yum源和修改内核参数（修改网卡）
<pre>
cobbler profile edit --name=CentOS-7-x86_64 --repos="openstack-mitaka" --distro=CentOS-7-x86_64  --kickstart=/var/lib/cobbler/kickstarts/CentOS-7-x86_64.cfg
cobbler profile edit --name=CentOS-7-x86_64 --kopts='net.ifnames=0 biosdevname=0'
cobbler profile report CentOS-7-x86_64
</pre>
* 修改cobbler界面提示
<pre>
vim /etc/cobbler/pxe/pxedefault.template
http://www.stanley.com
</pre>
* 生成配置文件，现在就可以进行开机半自动化了
<pre>
systemctl  restart cobblerd
systemctl  restart xinetd
cobbler sync
</pre>

* 如果想完全自动化，就需要指定mac地址
	
	例如
<pre>
cobbler system add --name=stanley --mac=00:50:56:28:44:32 --profile=CentOS-7-x86_64 --ip-address=192.168.56.111 --subnet=255.255.255.0 --gateway=192.168.56.2 --interface=eth0 --static=1 --hostname=linux.example.com --name-servers="114.114.114.114"
cobbler sync
</pre>
* [参考网站](http://blog.sina.com.cn/s/blog_61c07ac50101d074.html)

##Python自动添加
<pre>
s#!/usr/bin/env python 
# -*- coding: utf-8 -*-
import xmlrpclib 

class CobblerAPI(object):
    def __init__(self,url,user,password):
        self.cobbler_user= user
        self.cobbler_pass = password
        self.cobbler_url = url
    
    def add_system(self,hostname,ip_add,mac_add,profile):
        '''
        Add Cobbler System Infomation
        '''
        ret = {
            "result": True,
            "comment": [],
        }
        #get token
        remote = xmlrpclib.Server(self.cobbler_url) 
        token = remote.login(self.cobbler_user,self.cobbler_pass) 
		
		#add system
        system_id = remote.new_system(token) 
        remote.modify_system(system_id,"name",hostname,token) 
        remote.modify_system(system_id,"hostname",hostname,token) 
        remote.modify_system(system_id,'modify_interface', { 
            "macaddress-eth0" : mac_add, 
            "ipaddress-eth0" : ip_add, 
            "dnsname-eth0" : hostname, 
        }, token) 
        remote.modify_system(system_id,"profile",profile,token) 
        remote.save_system(system_id, token) 
        try:
            remote.sync(token)
        except Exception as e:
            ret['result'] = False
            ret['comment'].append(str(e))
        return ret

def main():
    cobbler = CobblerAPI("http://192.168.56.12/cobbler_api","cobbler","cobbler")
    ret = cobbler.add_system(hostname='cobbler-api-test',ip_add='192.168.56.111',mac_add='00:50:56:25:C2:AA',profile='CentOS-7-x86_64')
    print ret

if __name__ == '__main__':
    main()
</pre>

##Python打印列表
```	
#!/usr/bin/python
import xmlrpclib
server = xmlrpclib.Server("http://192.168.56.12/cobbler_api")
print server.get_distros()
print server.get_profiles()
print server.get_systems()
print server.get_images()
print server.get_repos()
```
* 执行结果
```
python cobbler_list.py 
     
[{'comment': '', 'kernel': '/var/www/cobbler/ks_mirror/CentOS-7-x86_64/images/pxeboot/vmlinuz', 'uid': 'MTQ2NDcxOTYxMi40ODY3NTM1ODUuMzUwOA', 'kernel_options_post': {}, 'redhat_management_key': '<<inherit>>', 'kernel_options': {}, 'redhat_management_server': '<<inherit>>', 'initrd': '/var/www/cobbler/ks_mirror/CentOS-7-x86_64/images/pxeboot/initrd.img', 'mtime': 1464719612.957186, 'template_files': {}, 'ks_meta': {'tree': 'http://@@http_server@@/cblr/links/CentOS-7-x86_64'}, 'boot_files': {}, 'breed': 'redhat', 'os_version': 'rhel7', 'mgmt_classes': [], 'fetchable_files': {}, 'tree_build_time': 0, 'arch': 'x86_64', 'name': 'CentOS-7-x86_64', 'owners': ['admin'], 'ctime': 1464719612.480652, 'source_repos': [['http://@@http_server@@/cobbler/ks_mirror/config/CentOS-7-x86_64-0.repo', 'http://@@http_server@@/cobbler/ks_mirror/CentOS-7-x86_64']], 'depth': 0}]
[{'comment': '', 'kickstart': '/var/lib/cobbler/kickstarts/CentOS-7-x86_64.cfg', 'name_servers_search': [], 'ks_meta': {}, 'kernel_options_post': {}, 'repos': ['openstack-mitaka'], 'redhat_management_key': '<<inherit>>', 'virt_path': '', 'kernel_options': {'biosdevname': '0', 'net.ifnames': '0'}, 'name_servers': [], 'mtime': 1464719706.748638, 'enable_gpxe': 0, 'template_files': {}, 'uid': 'MTQ2NDcxOTYxMi44NDQ4MTM1ODcuMzE1NQ', 'virt_auto_boot': 1, 'virt_cpus': 1, 'mgmt_parameters': '<<inherit>>', 'boot_files': {}, 'mgmt_classes': [], 'distro': 'CentOS-7-x86_64', 'dhcp_tag': 'default', 'virt_bridge': 'xenbr0', 'parent': '', 'virt_type': 'kvm', 'proxy': '', 'enable_menu': 1, 'fetchable_files': {}, 'virt_file_size': 5, 'ctime': 1464719612.84001, 'virt_disk_driver': 'raw', 'owners': ['admin'], 'name': 'CentOS-7-x86_64', 'virt_ram': 512, 'server': '<<inherit>>', 'redhat_management_server': '<<inherit>>', 'depth': 1, 'template_remote_kickstarts': 0}]
[]
[]
[{'comment': '', 'priority': 99, 'apt_dists': '', 'ctime': 1464719456.416682, 'parent': '', 'yumopts': {}, 'apt_components': '', 'breed': 'yum', 'createrepo_flags': '<<inherit>>', 'owners': ['admin'], 'mirror_locally': True, 'environment': {}, 'depth': 2, 'proxy': '', 'mtime': 1464719456.416682, 'uid': 'MTQ2NDcxOTQ1Ni40Mjk4MjE0NDkuMzE1OTE', 'keep_updated': True, 'arch': 'x86_64', 'mirror': 'http://mirrors.aliyun.com/centos/7.2.1511/cloud/x86_64/openstack-mitaka/', 'rpm_list': [], 'name': 'openstack-mitaka'}]
```

