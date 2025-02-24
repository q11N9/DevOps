# Zabbix
# I. What is Zabbix? 
Zabbix is an **open-source monitoring solution** used for tracking the status of IT infrastructure, including networks, servers, applications, and cloud services. 

It provides real-time monitoring, data collection, alerting, and visualization tools to help administrators maintain system health and performance

# II. How to install Zabbix
In this article, i will install Zabbix 6.4 on Ubuntu 20.04 Focal. You can see more in here: [Download Zabbix 6.4](https://www.zabbix.com/download?zabbix=6.4&os_distribution=ubuntu&os_version=20.04&components=server_frontend_agent&db=mysql&ws=apache)

![image](https://github.com/user-attachments/assets/15c92d9c-4ec2-4d78-9f35-05e78fc48654)

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

![image](https://github.com/user-attachments/assets/09972027-69e0-4af1-a11e-9467f88029a4)

Also on your windows:

![image](https://github.com/user-attachments/assets/91410e75-8a22-4de2-9294-607c80e93679)

Configure file `/etc/apache2/sites-available/000-default.conf` for not adding `/zabbix` each time you visit to your zabbix site.

![image](https://github.com/user-attachments/assets/c7da4e92-a7e7-479e-9914-664a9851870a)

And restart apache2 service:

```
systemctl restart apache2
```

It's worked perfectly now 

![image](https://github.com/user-attachments/assets/36c17045-a60a-49b1-9ea9-abd38bfee135)

Configure DB connection; 

![image](https://github.com/user-attachments/assets/231da9fb-31be-46af-9b4c-a8e5fdc62c5c)

Default database port is 3306 if you left it 0. 

Settings

![image](https://github.com/user-attachments/assets/3978ea2f-54de-4a28-ac05-1f9a1b03959a)

And finally login: 

![image](https://github.com/user-attachments/assets/90bc69e8-52a0-4640-b76c-7083f25fad72)

The default account is `Admin/zabbix`. 

![image](https://github.com/user-attachments/assets/45dab0b5-0d56-4928-883d-46487cf64053)

In **Monitoring -> Hosts**, there is a Zabbix agent we created before 

![image](https://github.com/user-attachments/assets/eb6b461d-de33-42a6-81ce-ef9e402c4ca2)

You can check it on your Zabbix server: 

![image](https://github.com/user-attachments/assets/241938bf-e54e-4117-a01b-ccb0cc23731a)

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

![image](https://github.com/user-attachments/assets/6efdbca0-2763-4d97-860c-ebcb1d9a6ecb)

![image](https://github.com/user-attachments/assets/163cd6e1-c77d-4c2e-8bbf-ac5cea56c10a)

It should work now 

![image](https://github.com/user-attachments/assets/6dd8a986-8c68-43f9-a3b2-f526e6a0d02c)

# III. Use Item and Trigger in Zabbix
If we want our shoeshop project somehow trigger our monitor, go to **Monitoring -> Hosts** an click into your hosts, choose **Items -> Create Item**

![image](https://github.com/user-attachments/assets/46af8558-b53d-424a-afde-73c695f36d70)

Configure it

![image](https://github.com/user-attachments/assets/6f72d3ae-2845-4670-9d62-919aa2fdc200)

Stop project and test:

![image](https://github.com/user-attachments/assets/def6ecdb-b016-42dd-84c1-15ea5af94445)

Now for trigger. Go to `Trigger` tag: 

![image](https://github.com/user-attachments/assets/20b63a41-8d04-430c-a14f-5f6433dc0101)

And create a new Trigger

![image](https://github.com/user-attachments/assets/da1b5605-728c-4010-891b-77afa3950d38)

Now we can saw the problem now 

![image](https://github.com/user-attachments/assets/87f0ebf6-587c-4d3f-b0b6-f3b1672b2ff4)

Using Jenkins to start project again, you can see it is resolved now 

![image](https://github.com/user-attachments/assets/1d29cfeb-6c7a-4efb-8d7e-9aaecb33cf23)
