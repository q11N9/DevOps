# Zabbix
# I. What is Zabbix? 
Zabbix is an **open-source monitoring solution** used for tracking the status of IT infrastructure, including networks, servers, applications, and cloud services. 

It provides real-time monitoring, data collection, alerting, and visualization tools to help administrators maintain system health and performance

# II. How to install Zabbix
In this article, i will install Zabbix 6.4 on Ubuntu 20.04 Focal. You can see more in here: [Download Zabbix 6.4](https://www.zabbix.com/download?zabbix=6.4&os_distribution=ubuntu&os_version=20.04&components=server_frontend_agent&db=mysql&ws=apache)

![image](https://hackmd.io/_uploads/HJ0EH-851x.png)

Install Zabbix repo: 

```bash!
$ wget https://repo.zabbix.com/zabbix/6.4/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_6.4+ubuntu20.04_all.deb
$ dpkg -i zabbix-release_latest_6.4+ubuntu20.04_all.deb
$ apt update
```

Install Zabbix server, frontend, agent

```bash!
$ apt install -y zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent
```

(If you dont want to monitor your zabbix server, remove `zabbix-agent` packet)

Install Mariadb Server 10.6

```bash!
$ apt install software-properties-common -y
$ curl -LsS https://r.mariadb.com/downloads/mariadb_repo_setup | sudo bash -s -- --mariadb-server-version="mariadb-10.6"
$ apt update -y
$ apt install -y mariadb-server-10.6 mariadb-client-10.6 mariadb-common
$ systemctl start mariadb
```

You can install different Mariadb in [here](https://mariadb.com/kb/en/mariadb-package-repository-setup-and-usage/). 

Create initial database: 

```sql!
$ mysql -uroot -p

mysql> create database zabbix character set utf8mb4 collate utf8mb4_bin;
mysql> create user zabbix@localhost identified by 'zabbix';
mysql> grant all privileges on zabbix.* to zabbix@localhost;
mysql> set global log_bin_trust_function_creators = 1;
mysql> quit;
```

Now we have an user `zabbix` with password `password`. 

On Zabbix server host import initial schema and data. You will be prompted to enter your newly created password.

```bash!
$ zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix
```

Disable log_bin_trust_function_creators option after importing database schema.

```sql!
mysql> set global log_bin_trust_function_creators = 0;
mysql> quit;
```

Edit file `/etc/zabbix/zabbix_server.conf`

```
DBPassword=zabbix
```

Start Zabbix proxy process and make it start at system boot.

```
$ systemctl restart zabbix-server zabbix-agent apache2
$ systemctl restart zabbix-server zabbix-agent apache2
```

Now, remember to modify your `/etc/hosts`:

![image](https://hackmd.io/_uploads/HkjTuzUcyg.png)

Also on your windows:

![image](https://hackmd.io/_uploads/HJJLYMI5yx.png)

Configure file `/etc/apache2/sites-available/000-default.conf` for not adding `/zabbix` each time you visit to your zabbix site.

![image](https://hackmd.io/_uploads/SJCTYGL9Je.png)

And restart apache2 service:

```
systemctl restart apache2
```

It's worked perfectly now 

![image](https://hackmd.io/_uploads/ryXrqM8qJl.png)

Configure DB connection; 

![image](https://hackmd.io/_uploads/HJL6of8qJe.png)

Default database port is 3306 if you left it 0. 

Settings

![image](https://hackmd.io/_uploads/S147nzIq1x.png)

And finally login: 

![image](https://hackmd.io/_uploads/BJWvnG89kl.png)

The default account is `Admin-zabbix`. 

![image](https://hackmd.io/_uploads/HyT_2GL91e.png)

In **Monitoring -> Hosts **, there is a Zabbix agent we created before 

![image](https://hackmd.io/_uploads/BkbypfI5Je.png)

You can check it on your Zabbix server: 

![image](https://hackmd.io/_uploads/B1aZpMLqkl.png)

Now if you want to monitor any servers, just install `zabbix-agent` on it. 

After installing, configure file `/etc/zabbix/zabbix_agentd.conf`: 

```
Server=zabbix.fucalors.tech
ServerActive=zabbix.fucalors.tech
Hostname=gitlab-server_192.168.240.100
```

Now restart zabbix agent service.

```bash!
systemctl restart zabbix-agent
```

In Zabbix, go to **Monitoring -> Hosts -> Create host**

![image](https://hackmd.io/_uploads/BJsBJ7I5Jx.png)

![image](https://hackmd.io/_uploads/HJDokmUqyg.png)

It should work now 

![image](https://hackmd.io/_uploads/BJ_yxXU9Jl.png)

# III. Use Item and Trigger in Zabbix
If we want our shoeshop project somehow trigger our monitor, go to **Monitoring -> Hosts** an click into your hosts, choose **Items -> Create Item**

![image](https://hackmd.io/_uploads/HyyKVXU5yx.png)

Configure it

![image](https://hackmd.io/_uploads/BkzNr7I5kx.png)

Stop project and test:

![image](https://hackmd.io/_uploads/rkxtDmL5yl.png)

Now for trigger. Go to `Trigger` tag: 

![image](https://hackmd.io/_uploads/B1eAPQ8c1e.png)

And create a new Trigger

![image](https://hackmd.io/_uploads/ryPrumU91e.png)

Now we can saw the problem now 

![image](https://hackmd.io/_uploads/rJc_um85ke.png)

Using Jenkins to start project again, you can see it is resolved now 

![image](https://hackmd.io/_uploads/r1rCdmL9Jl.png)
