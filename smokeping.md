#smokeping安装使用
##smokeping安装
安装环境

* 系统：`CentOS release 6.3 (Final)`
* 内核：`2.6.32-279.el6.x86_64`

----------
 

安装依赖和Apache
<pre>
rpm -Uvh http://apt.sw.be/redhat/el6/en/x86_64/rpmforge/RPMS/rpmforge-release-0.5.3-1.el6.rf.x86_64.rpm
yum -y install perl perl-Net-Telnet perl-Net-DNS perl-LDAP perl-libwww-perl perl-RadiusPerl perl-IO-Socket-SSL perl-Socket6 perl-CGI-SpeedyCGI perl-FCGI  perl-Time-HiRes perl-ExtUtils-MakeMaker perl-RRD-Simple rrdtool rrdtool-perl curl fping echoping  httpd httpd-devel gcc make  wget libxml2-devel libpng-devel glib pango pango-devel freetype freetype-devel fontconfig cairo cairo-devel libart_lgpl libart_lgpl-devel mod_fastcgi screen
rpm -ivh http://pkgs.repoforge.org/perl-CGI-SpeedyCGI/perl-CGI-SpeedyCGI-2.22-1.2.src.rpm
perl-CGI-SpeedCGI
</pre>
下载smokeping
<pre>
cd  /home/jifucha/tools/
wget -c http://oss.oetiker.ch/smokeping/pub/smokeping-2.6.11.tar.gz
tar zxf smokeping-2.6.11.tar.gz
cd smokeping-2.6.11
./setup/build-perl-modules.sh /usr/local/smokeping/thirdparty
./configure --prefix=/usr/local/smokeping && gmake install
</pre>
* 创建首页文件
<pre>
cp /usr/local/smokeping/htdocs/smokeping.fcgi.dist /usr/local/smokeping/htdocs/smokeping.fcgi
</pre>
* 创建配置文件
<pre>
cp /usr/local/smokeping/etc/config.dist /usr/local/smokeping/etc/config
</pre>
* 添加中文支持
<pre>
yum -y install wqy-zenhei-fonts.noarch
sed -i '49icharset = utf-8' /usr/local/smokeping/etc/config
</pre>
* 设置监控实例
<pre>
sed -i 's#cgiurl   = http://some.url/smokeping.cgi#cgiurl   = http://192.168.1.5/smokeping.cgi#g' /usr/local/smokeping/etc/config
sed -i 's/james.address/127.0.0.1/' /usr/local/smokeping/etc/config
sed -i 's/\/Test\/James \/Test\/James\~boomer/127.0.0.1/' /usr/local/smokeping/etc/config
</pre>
* 调整检测时间间隔和ping次数
<pre>
sed -i 's#300#60#g' /usr/local/smokeping/etc/config      
sed -i 's#20#60#g' /usr/local/smokeping/etc/config
</pre>
* 创建相关目录并设置相关权限
<pre>
egrep "^User |^Group " /etc/httpd/conf/httpd.conf  
mkdir -p /usr/local/smokeping/{data,cache,var}
chown apache.apache /usr/local/smokeping/{data,cache,var}
chmod 400 /usr/local/smokeping/etc/smokeping_secrets.dist
</pre>
* 修改Apache和生成网页加密
<pre>
htpasswd -c /usr/local/smokeping/htdocs/htpasswd  smokeping
</pre>
* 修改apache配置文件
<pre>
vim /etc/httpd/conf/httpd.conf 
</pre>
> 在此行下增加如下```DocumentRoot "/var/www/html"```

	Alias /cache "/usr/local/smokeping/cache/"
	Alias /cropper "/usr/local/smokeping/htdocs/cropper/"
	Alias /smokeping "/usr/local/smokeping/htdocs/smokeping.fcgi"
	<Directory "/usr/local/smokeping">
	AllowOverride None
	Options All
	AddHandler cgi-script .fcgi .cgi
	Order allow,deny
	Allow from all
	AuthName "smokeping"
	AuthType Basic
	AuthUserFile /usr/local/smokeping/htdocs/htpasswd
	Require valid-user
	DirectoryIndex smokeping.fcgi
	</Directory>
> 重启Apache
<pre>
/etc/init.d/httpd restart
</pre>
* 启动smokeping
<pre>
/usr/local/smokeping/bin/smokeping
</pre>
* web界面访问
> 192.168.1.5/smokeping
##汉化
<pre>
sed -i 's/The most interesting destinations/XX科技网络PING监控平台/' /usr/local/smokeping/etc/config
sed -i 's/ Network Latency Grapher/XX科技网络PING监控平台/' /usr/local/smokeping/etc/config
sed -i 's/Welcome to the SmokePing website of xxx Company./欢迎光临XX科技公司SmokePing网站。/' /usr/local/smokeping/etc/config
sed -i 's/Here you will learn all about the latency of our network./在这里，您将了解到所有关于我们的网络延迟。/' /usr/local/smokeping/etc/config
sed -i 's/Top Standard Deviation/标准偏差/' /usr/local/smokeping/etc/config
sed -i 's/Top Max Roundtrip Time/最大延迟/' /usr/local/smokeping/etc/config
sed -i 's/Top Median Roundtrip Time/平均延迟/' /usr/local/smokeping/etc/config
sed -i 's/Top Packet Loss/丢包率/' /usr/local/smokeping/etc/config
sed -i 's/Last 3 Hours/最后3 小时/' /usr/local/smokeping/etc/config
sed -i 's/Last 30 Hours/最后 30 小时/' /usr/local/smokeping/etc/config
sed -i 's/Last 10 Days/最后 10 天/' /usr/local/smokeping/etc/config
sed -i 's/Last 400 Days/最后 400 天/' /usr/local/smokeping/etc/config
sed -i 's/median rtt/平均延迟/' /usr/local/smokeping/lib/Smokeping.pm
sed -i 's/:packet loss/:丢包率/' /usr/local/smokeping/lib/Smokeping.pm
sed -i 's/loss color/丢包颜色/' /usr/local/smokeping/lib/Smokeping.pm
sed -i 's/T:probe/T:探测器/' /usr/local/smokeping/lib/Smokeping.pm
sed -i 's/:end/:结束时间/' /usr/local/smokeping/lib/Smokeping.pm
sed -i 's/Navigator Graph/导航图/' /usr/local/smokeping/lib/Smokeping.pm
sed -i 's/SmokeAlert/XX Smoke 报警/' /usr/local/smokeping/lib/Smokeping.pm
</pre>


##报错
###打开web界面报错
<pre>
Software error:
Version mismatch between Carp 1.11 (/usr/share/perl5/Carp.pm) and Carp::Heavy 1.38 (/usr/local/smokeping/bin/../thirdparty/lib/perl5/Carp/Heavy.pm).  Did you alter @INC after Carp was loaded?
Compilation failed in require at (eval 19) line 2.
For help, please send mail to the webmaster (root@localhost), giving this error message and the time and date of the error.
Software error:
[Fri Jun 17 09:15:50 2016] Heavy.pm: Version mismatch between Carp 1.11 (/usr/share/perl5/Carp.pm) and Carp::Heavy 1.38 (/usr/local/smokeping/bin/../thirdparty/lib/perl5/Carp/Heavy.pm).  Did you alter @INC after Carp was loaded?
[Fri Jun 17 09:15:50 2016] Heavy.pm: Compilation failed in require at (eval 19) line 2.
BEGIN failed--compilation aborted at (eval 19) line 2.
For help, please send mail to the webmaster (root@localhost), giving this error message and the time and date of the error.
</pre>
* 解决方法
<pre>
\cp /usr/share/perl5/Carp.pm /usr/local/smokeping/bin/../thirdparty/lib/perl5/Carp/Heavy.pm
\cp /usr/share/perl5/Carp.pm /usr/local/smokeping/bin/../thirdparty/lib/perl5/Carp/Heavy.pm
</pre>
###启动报错
<pre>
Error: RRD parameter mismatch ('Wrong value of step: /usr/local/smokeping/data/Test/James.rrd has 60, create string has 300'). You must delete /usr/local/smokeping/data/Test/James.rrd or fix the configuration parameters.
</pre>
* 解决方法
<pre>
rm /usr/local/smokeping/data/Test/* -f
</pre>
###启动报错
<pre>
ERROR: /usr/local/smokeping/bin/../etc/config, line 112: File '/usr/local/smokeping/etc/smokeping_secrets.dist' is world-readable or writable, refusing it
</pre>
* 解决方法
<pre>
chmod 400 /usr/local/smokeping/etc/smokeping_secrets.dist
</pre>
































