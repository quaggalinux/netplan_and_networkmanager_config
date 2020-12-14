# netplan_and_networkmanager_config  
ubuntu使用netplan配置及改回network-manager配置方法  
  
前几天有个玩家问我ubuntu18怎么改DNS地址,而且重启也可以保持,我一听就知道这货奔那个飞去了,幸好之前搞过IPv6小鸡,磨了DNS设置这把砍柴刀很久了,那就砍它母鸡的(∩•̀ω•́)⊃-*⋆  
  
由于ubuntu18开始改用netplan控制网络,所以DNS设置这个方法就变成各庙各菩萨了,下面有3个方法,第一个方法是野路子,适合大干快上的玩家,我个人是不赞成的,但也分享出来  
   
# 第一种方法   
   
------全局DNS设置(不建议,因为是野路子,正路方法是搞netplan或network-manager)------   
   
修改dns解析文件   
   
$su   
#cd /  
#nano /etc/resolv.conf  
   
锁定resolv.conf文件不让系统重启时修改的方法，先改好resolv.conf文件内容，然后按下面步骤操作  
   
#cp /etc/resolv.conf /tmp/resolv.conf   
#rm /etc/resolv.conf  
#mv /tmp/resolv.conf /etc/resolv.conf  
#chattr +i /etc/resolv.conf  
  
解除文件锁定  
#chattr -i /etc/resolv.conf  
  
查DNS状态，按键盘下箭号翻页看物理接口的DNS地址，q键退出  
#systemd-resolve --status  
  
Global显示的结果并不能说明当前使用的DNS服务器就是114.114.114.114和8.8.8.8，因为你修改/etc/systemd/resolved.conf下面的DNS=只影响Global，对于具体的网络接口所用的DNS服务器，要翻页到下面的Link N（网络接口名）部分才有  
  
# 第二种方法  
  
------按照正常手段修改netplan配置------   
   
注意:netplan修改完成后执行 netplan try 等待几秒钟，如果屏幕的读秒倒计时一直在动，说明修改没问题，接着回车即可(尽量不要 netplan apply，一旦修改错误你就再也连不上了!!!!!!!!!),修改netplan配置前建议在终端拷贝原来的配置文件文本并备份到本机,也没有几行的    
  
$su  
#cd /  
#nano /etc/netplan/00-installer-config.yaml （这个文件名可能每个系统不一样，具体进目录看）  
  
首先netplan现阶段的dhcp字段规则是以dhcp4字段优先决定,即当dhcp4: yes设置时,dhcp6设置什么都不起作用,照样能够拿到IPv6地址,但dhcp4: no设置时,dhcp6可以设置成dhcp6: yes,如果这时不设置IPv4静态地址,则接口只拿到IPv6地址  
  
"overrides"字段虽然有两个写法"dhcp4-overrides"及"dhcp6-overrides",其实只要写一个字段就可以了,而且无论写那个字段名都可以匹配dhcp4或dhcp6字段的设置状况,系统根本不管,但是如果两个字段都写的话反而要两种IP协议的"overrides"项都是一样的参数,否则netplan try时会报下面的错误信息  
"networkd requires that use-dns has the same value in both dhcp4_overrides and dhcp6_overrides"  
  
下面的样板配置是不给接口分配IPv4地址,这种情况下如果不设置静态IPv4地址,等于是接口只有IPv6地址,但反过来就不行了,只要dhcp4: yes设置时,无论如何都能拿到IPv6地址,前提是DHCP服务器有IPv6地址的分配  
  
无论接口是否有IPv4或IPv6地址,我们都可以同时设置DNS服务器为IPv4及IPv6地址  
  
以下内容由于使用空格缩进,而使用了全角输入空格,所以拷贝文本后要把空格重新在英文状态下输入一次!!!!!!!!!

network:  
　version: 2  
　renderer: networkd  
　　ethernets:  
　　　eth0:  
　　　　dhcp4: no  
　　　　dhcp6: yes  
　　　　dhcp6-overrides:  
　　　　　use-dns: no  
　　　　nameservers:  
　　　　　search: [local,node]  
　　　　　addresses: [202.59.114.100, 202.59.114.143, "2001:4860:4860::8888"]  
  
另外是一般情况下我们改DNS的正常样板配置,因为已经用了dhcp4: yes设置,所以dhcp6设置可以省略了  
  
先用#systemd-resolve --status命令记录下运营商原先的DNS地址,按键盘下箭号翻页看物理接口的DNS地址,一般在Link项目下,然后按照下面的样板配置,更改其中的IPv4或IPv6的DNS地址,或两种都更改,地址个数是1/2/3/4个都无所谓,当然如果系统缺省的netplan配置文件是DHCP方式的,那么就不会写运营商的DNS服务器地址在里面,所以我们可以按照上面的命令记录下运营商原先的DNS地址,按照自己的需求组合好填上去,但关键是按照下面的填写规则  
  
DNS的IP地址之间要用","号分隔,每个","号后面要跟空格,IPv6地址必须必须用""符号引用,任何一项不满足会报错,但netplan try不告诉你具体原因让你无法查找,一般建议先写IPv4地址然后再写IPv6地址  
  
search:项没有特殊要求的话可以随便写一项或两项,如果只写前面一项的话,系统一般会自动加回运营商的search domain到第二项,这个是配置完成并enter键接受后再用#systemd-resolve --status命令可以看到  
  
而且一定要注意子项缩进两个空格,同等级别子项必须对齐,任何一项不满足会报错,但netplan try不告诉你具体原因让你无法查找  
  
network:  
  version: 2  
  renderer: networkd  
  ethernets:  
    ens33:  
      dhcp4: yes  
      dhcp4-overrides:  
        use-dns: no  
      nameservers:  
        search: [internet]  
        addresses: [202.59.114.100, 202.59.114.143, "2001:4860:4860::8888"]  
  
一般系统安装后的netplan配置,已经同时支持IPv6的DHCP请求  
network:  
  ethernets:  
    ens33:  
      dhcp4: true  
  version: 2  
  
 # 第3种方法
 
------ubuntu 18或以上版本改回用原来的network-manager进行网络设置------  
  
基于netplan的功能不全或bug,举例就是不能控制IPv4及IPv6在DHCP情况下其中一种IP协议自定义DNS服务器地址，以及IPv4是DHCP而IPv6想关闭DHCP,所以需要用以下方法启用network-manager进行网络设置  
  
$su  
#cd /  
  
以超级用户安装NetworkManager  
  
#apt install network-manager -y  
  
以超级用户重新启动 NetworkManager 服务  
  
#systemctl start NetworkManager.service  
  
#systemctl enable NetworkManager  
  
#nano /etc/netplan/00-installer-config.yaml （这个文件名可能每个系统不一样，具体进目录看）  
  
改成以下的样子，就可以让NetworkManager控制网络设置  
  
network:  
  renderer: NetworkManager  
  ethernets:  
    ens33:  
      dhcp4: true  
  version: 2  
  
保存后退出，执行 netplan try 等待几秒钟，如果屏幕的读秒倒计时一直在动，说明修改没问题，可以按回车接受新配置  
  
以超级用户执行下面的菜单命令程序运行NetworkManager网络设置  
  
#nmtui  
  
进去后设置好主要的以太网卡，如禁止IPv4，IPv6只是获得IP，手动设置IPv6的DNS64服务器地址等，菜单设置样板可参考截图，确认退出后IP地址会被更新，所以Xshell的连接会中断，然后要用新的IP地址连接，重新连接后可以再运行nmtui程序，删除netplan的接口网卡及其他与主以太网卡同名字的网卡  
  






