#1.基础套件安装
- 检查基础套件是否安装

     	rpm -q gcc glibc glibc-common gd gd-devel xinetd openssl-devel

- 根据检测的结果，安装没有的套件

    	yum install -y  套件名            
    
#2.安装nagios及其插件

	    yum install -y nagios  nagios-plugins-*

#3.安装Apache及PHP
用yum方式安装nagios及其插件时，apache和php已经安装完成，可通过

	rpm -qa php httpd

来查看php和apache是否已经安装，若未安装通过yum安装

	yum install -y httpd  php
#4.修改nagios网页登陆用户及密码
nagios默认用户名为nagiosadmin，密码为nagiosadmin，若想修改密码，操作如下：

	htpasswd -C /etc/nagios/passwd nagiosadmin


启动nagios及httpd:

	service nagios start
	service httpd start

访问nagios页面URL为：

	http://IP地址/nagios/

若在访问nagios页面登陆不上，或能登陆但是页面没有数据，修改nagios相关文件用户及用户组即可解决，nagios用户及用户组在安装nagios时自动创建的：

	chown -R nagios.nagios /usr/share/nagios
	chown -R nagios.nagios /etc/nagios

#5.配置nagios
- 创建hosts.cfg和services.cfg

hosts.cfg用来定义被监控的主机及主机组

services.cfg用来定义服务，此文件里定义的是需要监控的服务

创建及编辑host.cfg

	cd /etc/nagios/objects
	vim hosts.cfg

	define   host{
         use              linux-server  #引用主机linux-server的属性信息，linux-server主机在templates.cfg文件中进行了定义。
         host_name        igWeb39    #主机名
         alias            igWeb39    #主机别名
         address          172.19.0.39#被监控主机的IP
  	 }
	define   host{
         use              linux-server  #引用主机linux-server的属性信息，linux-server主机在templates.cfg文件中进行了定义。
         host_name        igWeb38    #主机名
         alias            igWeb38    #主机别名
         address          172.19.0.38#被监控主机的IP
  	 }
	//定义一个主机组   
	define hostgroup{      
        hostgroup_name          bsmart-servers        #主机组名称，可以随意指定。
        alias                   bsmart servers        #主机组别名
        members                 igWeb39,igWeb38       #主机组成员,这里调用的是define host里已经定义的主机，每个主机间用逗号隔开  
        }

创建及编辑services.cfg

	vim /etc/nagios/objects/services.cfg

	define service{
        use                     local-service  #引用local-service服务的属性值，local-service在templates.cfg文件中进行了定义。
        host_name               igWeb39         #指定要监控哪个主机上的服务，igweb39在hosts.cfg文件中进行了定义。若同时监控多个主机同样的服务，可修改为：hostgroup_name   bsmart-servers
        service_description     Current Load
        check_command           check_nrpe!check_load #指定监控命令，
        }

将hosts.cfg和services.cfg添加到nagios.cfg中

	cd /etc/nagios/
	vim nagios.cfg

	cfg_file=/etc/nagios/objects/hosts.cfg
	cfg_file=/etc/nagios/objects/services.cfg

在commands.cfg中增加对check_nrpe的定义

	cd /etc/nagios/objects
	vim commands.cfg 

	define command{
       command_name    check_nrpe  # 定义命令名称为check_nrpe,在services.cfg中要使用这个名称.
       command_line    $USER1$/check_nrpe  -H $HOSTADDRESS$  -c  $ARG1$  #这是定义实际运行的插件程序

       }



验证nagios配置文件

	/usr/sbin/nagios -v /etc/nagios/nagios.cfg 

重新加载nagios

	service nagios reload

#6.被监控主机配置
将监控服务端IP添加到被监控机白名单

	vim /etc/nagios/nrpe.cfg 

	allowed_hosts=127.0.0.1,(服务端IP)

添加要监控的服务，这里定义的服务监控主机里nagios配置文件/etc/nagios/objects/services.cfg调用，

	check_command   check_nrpe!check_load
check_load即在被监控主机/etc/nagios/nrpe.cfg定义的

	cd etc/nagios/
	vim nrpe.cfg

	command[check_load]=/usr/lib64/nagios/plugins/check_load -w 15,10,5 -c 30,25,20
	command[check_home]=/usr/lib64/nagios/plugins/check_disk -w 20% -c 10% -p /dev/mapper/VolGroup-lv_home
要监控的服务需要添加到此文件

#7.报警邮件设置
当监控到异常情况时，将信息发送到指定邮箱

安装sendmail组件

	yum install -y sendmail*
	service sendmail start

修改vim /etc/nagios/nagios.cfg

	enable_notifications=1          #开启后也就是nagios装的所有插件，出现问题都会报警


添加联系人到nagios配置文件

	cd /etc/nagios/objects
	vim contacts.cfg 

	define contact{
        contact_name                    liulei            #联系人的名称,这个地方不要有空格
        use                             generic-contact   #引用generic-contact的属性信息，其中“generic-contact”在templates.cfg文件中进行定义
        alias                           liulei
        email                           liulei@iot.com
        }
	define contact{
        contact_name                    leiliu            #联系人的名称,这个地方不要有空格
        use                             generic-contact   #引用generic-contact的属性信息，其中“generic-contact”在templates.cfg文件中进行定义
        alias                           leiliu
        email                           leiliu@iot.com
        }
	define contactgroup{
        contactgroup_name       ctgroup              #联系人组的名称,同样不能空格
        alias                   ctgroup              #联系人组描述
        members                 liulei,leiliu        #联系人组成员，其中“david”就是上面定义的联系人，如果有多个联系人则以逗号相隔
        }