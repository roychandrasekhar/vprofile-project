# Setup multi stack Tomcat project in Vagrant

### Provisioning of the VM should be in this order
1. MySQL (Database)
2. Memcache (DB Caching)
3. RabbitMQ (Broker/Queue)
4. Tomcat (Application)
5. Nginx (Web Server)



---


1. Run the vagrant up command to initialize all the VM with there static IP. Please check the Vagrant file in the source
(I am running this in my windows machine)
    
    Additionally set host file entry in every VM
    ```bash
    cat /etc/hosts
        ...
        192.168.56.15   db01
        192.168.56.14   mc01
        192.168.56.13   rmq01
        192.168.56.12   app01
        192.168.56.11   web01
    ```


2. Setup MySQL Databse and import the DB
    * The multi setup Vagrant file should only setup the Centos VM. MySQL should be install by SSH into the machine
        - Other installation will be directly on the Machine by SSH into it
        - Set the DB password as 'admin123'
            ```bash 
           # update the hosts file as above
           sudo mysql_secure_installation
           # Set root password [Y/n] Y
           # Remove anonymous users [Y/n] Y
           # Disallow root login remotely [Y/n] n
           # Remove test database and access to it? [Y/n] Y
           # Reload priviliege tables now? [Y/n] Y
           
           mysql -u root -padmin123
           ```
           
           Create the DB and account
           ```sql
           create database accounts;
           grant all privileges on accounts.* TO 'admin'@'%' identified by 'admin123';
            FLUSH PRIVILEGES;
          exit;
           ```
           
           Import the source into DB
           ```bash
           git clone -b main https://github.com/roychandrasekhar/vprofile-project.git
           cd vprofile-project
           mysql -u root -padmin123 accounts < src/main/resources/db_backup.sql
           mysql -u root -padmin123 accounts
           ```
           
           ```sql
           show tables;
           exit;
           ```
           
           Set the firewall and open 3306 port
           ```bash
           sudo systemctl start firewalld
           sudo systemctl enable firewalld
           sudo firewall-cmd --get-active-zones
           sudo firewall-cmd --zone=public --add-port=3306/tcp --permanent
           sudo firewall-cmd --reload
           sudo systemctl restart mariadb
           ```
3. Now setup the MEMCACHE server
        - SSH into `mc01`
        - Update the hosts file as above
        - start & enable memcache on port 11211

    ```bash
    sed -i 's/127.0.0.1/0.0.0.0/g' /etc/sysconfig/memcached
    
    # Open firewall memcache
    sudo systemctl start firewalld
    sudo systemctl enable firewalld
    sudo firewall-cmd --add-port=11211/tcp
    sudo firewall-cmd --runtime-to-permanent
    sudo firewall-cmd --add-port=11111/udp
    sudo firewall-cmd --runtime-to-permanent
    sudo memcached -p 11211 -U 11111 -u memcached -d
    sudo firewall-cmd --reload
    sudo systemctl restart memcached
    ```
    
4. Now setup the RabbitMQ server
        - SSH into `rmq01`
        - Update the hosts file as above
        - Setup access to user test and make it admin
    ```bash
    sudo sh -c 'echo "[{rabbit, [{loopback_users, []}]}]." > /etc/rabbitmq/rabbitmq.config'
    sudo rabbitmqctl add_user test test
    sudo rabbitmqctl set_user_tags test administrator
    sudo systemctl restart rabbitmq-server
    ```

    Starting the firewall and allowing the port 5672 to access rabbitmq
    ```bash
    sudo systemctl start firewalld
    sudo systemctl enable firewalld
    sudo firewall-cmd --add-port=5672/tcp
    sudo firewall-cmd --runtime-to-permanent
    sudo systemctl start rabbitmq-server
    sudo systemctl enable --now rabbitmq-server
    ```

5. Now setup the Tomcat server
        - SSH into `app01`
        - Update the hosts file as above
        - Download the tomact and install
    ```bash
    cd /tmp/
    wget https://archive.apache.org/dist/tomcat/tomcat-9/v9.0.75/bin/apache-tomcat-9.0.75.tar.gz
    tar xzvf apache-tomcat-9.0.75.tar.gz
    useradd --home-dir /usr/local/tomcat --shell /sbin/nologin tomcat
    cp -r /tmp/apache-tomcat-9.0.75/* /usr/local/tomcat/
    chown -R tomcat.tomcat /usr/local/tomcat
    ```
    
    Now make the Tomcat as service and run
    ```bash
    sudo vi /etc/systemd/system/tomcat.service
        ...
        [Unit]
        Description=Tomcat
        After=network.target

        [Service]
        WorkingDirectory=/usr/local/tomcat
        Environment=JRE_HOME=/usr/lib/jvm/jre
        Environment=JAVA_HOME=/usr/lib/jvm/jre
        Environment=CATALINA_HOME=/usr/local/tomcat
        Environment=CATALINA_BASE=/usr/local/tomcat
        ExecStart=/usr/local/tomcat/bin/catalina.sh run
        ExecStop=/usr/local/tomcat/bin/shutdown.sh run
        SyslogIdentifier=tomcat-%i

        [Install]
        WantedBy=multi-user.target
    ```
    
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl start tomcat
    sudo systemctl enable --now tomcat
    ```
    
    Enabling the firewall and allowing port 8080 to access the tomcat
    ```bash
    sudo systemctl start firewalld
    sudo systemctl enable firewalld
    sudo firewall-cmd --get-active-zones
    sudo firewall-cmd --zone=public --add-port=8080/tcp --permanent
    sudo firewall-cmd --reload
    ```
    
    Pull the code and deploy it
    ```bash
    git clone -b main https://github.com/roychandrasekhar/vprofile-project.git
    cd vprofile-project
    # update the property file if there is anything 
    nano src/main/resources/application.properties
    ```
    
    Now build the code and deploy
    ```bash
    mvm install
    sudo systemctl stop tomcat
    sudo rm -rf /usr/local/tomcat/webapps/ROOT*
    sudo cp target/vprofile-v2.war /usr/local/tomcat/webapps/ROOT.war
    sudo chown tomcat.tomcat /usr/local/tomcat/webapps -R
    sudo systemctl stop tomcat
    sudo systemctl restart tomcat
    sudo systemctl enable --now tomcat
    ```
6. Finally setup the Nginx server (or load balancer)
        - SSH into `web01`
        - Update the hosts file as above 
        - Create Nginx conf file
    ```bash
    sudo rm -rf /etc/nginx/sites-enabled/default
    sudo rm -f /etc/nginx/sites-available/default
    sudo nano /etc/nginx/sites-available/vproapp
        ...
        server {
            listen 80;
            location / {
                proxy_pass http://vproapp;
            }
        }
        upstream vproapp {
            server app01:8080;
        }
      
    # verify the configuration
    nginx -t
    sudo ln -s /etc/nginx/sites-available/vproapp /etc/nginx/sites-enabled/vproapp
    sudo systemctl restart nginx
    ```
7. Now check from your browser this URL
    http://web01
    ![](https://i.imgur.com/67vOEg0.png0)
    Username : admin_vp
    Pass : admin_vp
    
    
    ![](https://i.imgur.com/bVDOWzL.png)
