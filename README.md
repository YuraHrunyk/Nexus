# Nexus
In this tutorial we will try to install Nexus Repository Manager OSS 3.2.0-01
1. At first we need to download latest version of Nexus Repository Manager:

wget http://download.sonatype.com/nexus/3/latest-unix.tar.gz

2. Move the archive to /usr/local directory:

mv latest-unix.tar.gz/ /usr/local/
cd /usr/local

3. Extract it:

tar xvzf latest-unix.tar.gz

4. Then we need to install Java Virtual Machine. The version of the JVM must be at least 1.8 and at most 1.8:

cd /opt/

5. Download latest Java SE Development Kit 8 release and extract it:

wget --no-cookies --no-check-certificate --header "Cookie: gpw_e24=http%3A%2F%2Fwww.oracle.com%2F; oraclelicense=accept-securebackup-cookie" "http://download.oracle.com/otn-pub/java/jdk/8u111-b14/jdk-8u111-linux-x64.tar.gz"
tar xzf jdk-8u111-linux-x64.tar.gz

6. After extracting archive file use alternatives command to install it. alternatives command is available in chkconfig package:

cd /opt/jdk1.8.0_111/
alternatives --install /usr/bin/java java /opt/jdk1.8.0_111/bin/java 2
alternatives --config java
    There is 1 program that provides 'java'.
    Selection    Command
    *+ 1         /opt/jdk1.8.0_111/bin/java
    Enter to keep the current selection[+], or type selection number: 1

7. At this point JAVA 8 has been successfully installed on your system

alternatives --install /usr/bin/jar jar /opt/jdk1.8.0_111/bin/jar 2
alternatives --install /usr/bin/javac javac /opt/jdk1.8.0_111/bin/javac 2
alternatives --set jar /opt/jdk1.8.0_111/bin/jar
alternatives --set javac /opt/jdk1.8.0_111/bin/javac

8. Check the installed Java version on your system using following command:

java -version
    java version "1.8.0_111"
    Java(TM) SE Runtime Environment (build 1.8.0_111-b14)
    Java HotSpot(TM) 64-Bit Server VM (build 25.111-b14, mixed mode)

9. Setup JAVA_HOME Variable:

export JAVA_HOME=/opt/jdk1.8.0_111

10. Setup JRE_HOME Variable:

export JRE_HOME=/opt/jdk1.8.0_111/jre

11. Setup PATH Variable:

export PATH=$PATH:/opt/jdk1.8.0_111/bin:/opt/jdk1.8.0_111/jre/bin

12. Move to the bin folder and run the generic startup scripts called nexus and start the repository manager use the next commands:

cd /usr/local/nexus-3.2.0-01/bin
./nexus run

13. Before running the service configure an absolute path for your repository manager files. Then create a nexus user with sufficient access rights to run the service:

vim .bashrc
    NEXUS_HOME="/usr/local/nexus-3.2.0-01"

14. In bin/nexus.rc assign the user between the quotes in the line below:

vim bin/nexus.rc
    run_as_user="root"

15. If you use init.d instead of systemd, symlink $NEXUS_HOME/bin/nexus to /etc/init.d/nexus:

ln -s $NEXUS_HOME/bin/nexus /etc/init.d/nexus

16. This example uses chkconfig, a tool that targets the initscripts in init.d to run the nexus service. Run these commands to activate the service:

cd /etc/init.d
chkconfig --add nexus
chkconfig --levels 345 nexus on
service nexus start

17. This example is a script that uses systemd to run the repository manager service. Create a file called nexus.service. Add the following contents, then save the file in the /etc/systemd/system/ directory.

vim /etc/systemd/system/nexus.service
    [Unit]
    Description=nexus service
    After=network.target

    [Service]
    Type=forking
    ExecStart=/usr/local/nexus-3.2.0-01/bin/nexus start
    ExecStop=/usr/local/nexus-3.2.0-01/bin/nexus stop
    User=root
    Restart=on-abort

    [Install]
    WantedBy=multi-user.target

18. Activate the service with the following commands:

systemctl daemon-reload
systemctl enable nexus.service
systemctl start nexus.service

19. Then let's try to move to Nexus UI http://192.168.103.140:8081
but before this we should disable SELinux:

vim /etc/selinux/config
    # This file controls the state of SELinux on the system.
    # SELINUX= can take one of these three values:
    #     enforcing - SELinux security policy is enforced.
    #     permissive - SELinux prints warnings instead of enforcing.
    #     disabled - No SELinux policy is loaded.
    SELINUX=permissive
    # SELINUXTYPE= can take one of three two values:
    #     targeted - Targeted processes are protected,
    #     minimum - Modification of targeted policy. Only selected processes are protected.
    #     mls - Multi Level Security protection.
    SELINUXTYPE=targeted

20. Switch off firewall:

iptables -F
systemctl disable firewalld

And now we can use Nginx as Reverse Proxy on Restricted Ports.
21. Install nginx:

yum install -y nginx

22. Change nginx.conf configuration file in /etc/nginx/:

vim /etc/nginx/nginx.conf
    # For more information on configuration, see:
    #   * Official English Documentation: http://nginx.org/en/docs/
    #   * Official Russian Documentation: http://nginx.org/ru/docs/

    user nginx;
    worker_processes auto;
    error_log /var/log/nginx/error.log;
    pid /run/nginx.pid;

    # Load dynamic modules. See /usr/share/nginx/README.dynamic.
    include /usr/share/nginx/modules/*.conf;

    events {
        worker_connections 1024;
    }

    http {

        proxy_send_timeout 120;
        proxy_read_timeout 300;
        proxy_buffering    off;
        keepalive_timeout  5 5;
        tcp_nodelay        on;

    server {
        listen   *:80;
        server_name  www.nexus-artifactory.com;

        # allow large uploads of files
        client_max_body_size 1G;

        # optimize downloading files larger than 1G
        #proxy_max_temp_file_size 2G;

     location / {
        proxy_pass http://localhost:8081/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
      }
    }

23. Restart nginx service:

systemctl restart nginx

24. Well done! Try to move to Nexus UI on http://192.168.103.140:8081 
