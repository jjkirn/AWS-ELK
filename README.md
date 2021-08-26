# Install AWS OSS version of Elasticsearch on Ubuntu 20.04 (Desktop)

> This repository lists steps to create your own OSS Elasticsearch web application that collects syslog messages. The company [Elastic](https://elastic.co) has decided to no longer offer Elasticsearch (Elasticsearch, Kibana) under the Apache License, Version 2.0 (Alv2). This means that Elasticsearch and Kibana will no longer be open source software as of version 7.13 and beyond. As a result, AWS decided to create and maintain a fork of Elasticsearch referred to as [Open Distro Elasticsearch](https://opendistro.github.io/for-elasticsearch).

## What is OSS Elasticsearch
OSS stands for Open Source Software. An Apache 2.0-licensed distribution of Elasticsearch enhanced with enterprise security, alerting, SQL, and more was created by AWS. It consists of OSS Elasticsearch and Kibana (see reference 2 below). They are designed to work in tandem and it is relatively easy to get them setup on your Ubuntu system.

The below steps are designed to help you create your own Elasticsearch application designed to collect and search syslog messages. The application uses latest version Ubuntu 20.04 as the base (see reference 1 below). The application is created as a docker-compose application.

Logstash is not part of the OSS Elasticsearch distribution. However, there still is a OSS version of logstash (see reference 3. below) and that version is what is used in this application.

## Prerequisites
- Ubuntu 20.04 Desktop

## Install Steps
- [1. Download ISO](#Download ISO)
- [2. Create VM](#Create VM)
- [3. Setup Static IP](#Setup Static IP)
- [4. Install Docker](#Install Docker)
- [5. Install docker-compose](#Install docker-compose)
- [6. Create docker-compose.yml File]()
- [7. Create nginx.conf File]()
- [8. Create logstash.conf File]()
- [9. Get CA Signed Server certificate info for SSL install]()
- [10. Launch the docker-compose application]()
- [11. Sign in to Kibana](#Sign in to Kibana)

### Download ISO
**Step 1:** The Ubuntu 20.04 LTS ISO can downloaded at this [link](https://releases.ubuntu.com/focal/).

### Create VM
**Step 2:** I will be creating a VM in my VMware vSphere Cluster. The VM consisted of the following settings:
- 2 vCPUs
- 8 GB RAM
- 60 GB vDisk (or larger)
- VM installed on same network as your source of logs to ingest.

Use the downloaded ISO from Step 1 to create the VM. Then open a terminal window and do an update and upgrade:
```console
sudo apt update
sudo apt upgrade
```
### Setup Static IP
**Step 3:** To configure a static IP address on your Ubuntu 20.04 VM you need to modify a relevant netplan network configuration file within /etc/netplan/ directory. In my case the file's name is 01-network-manager-all.yaml. See below for the contents of this file:
```console
# This file describes the network interfaces available on your system
# For more information, see netplan(5).
network:
  version: 2
  renderer: networkd
  ethernets:
    ens160:
     dhcp4: no
	 dhcp6: no
     addresses: [192.168.1.89/24]
     gateway4: 192.168.1.1
     nameservers:
       addresses: [192.168.1.248,192.168.1.249]
```
> **Note:** You will need to modify/create this file to match the settings needed for your system.

Once you are satisfied with the contents of this file you can apply the settings to your system by executing the following:
```console
sudo netplan apply
```
In case you run into issues with netplan you can get further information by running:
```console
sudo netplan --debug apply
```

### Install Docker
**Step 4:** Begin by removing any old version.  Run the following to remove any old versions of docker:
```console
sudo apt-get remove docker docker-engine docker.io containerd runc
```
Run the following to enable apt to use a repository over HTTPS:
```console
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```
Now add Docker's offical GPG key:
```console
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```
Use the following command to set up the stable repository. 
```console
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
Install Docker Engine:
```
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```
Verify that the Docker Engine is installed correctly by running the hello-world image:
```
sudo docker run hello-world
```
### Install docker-compose
**Step 5:** You're now ready to install docker-compose. Run this command to download the current stable release of Docker Compose:

```console
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

> **Note:** Elasticsearch needs a high mmap count, which must be set (on the host!) with:
```console
sudo sysctl -w vm.max_map_count=262144
```

### Create docker-compose.yml File
**Step 6:** You're now ready to configure docker-compose. Create the file docker-compose.yml as follows:

```console
version: '3.7'
services:
  odfe-node1:
    image: amazon/opendistro-for-elasticsearch:1.13.2
    container_name: odfe-node1
    environment:
      - node.name=odfe-node1
      - discovery.type=single.node
      - bootstrap.memory_lock=true # along with the memlock settings below, disables swapping
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g" # minimum and maximum Java heap size, recommend setting both to 50% of system RAM
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536 # maximum number of open files for the Elasticsearch user, set to at least 65536 on modern systems
        hard: 65536
    volumes:
      - odfe-data1:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
      - 9600:9600 # required for Performance Analyzer
	healthcheck:
      test: ["CMD-SHELL", "curl --silent --fail logcalhost:9200:/_cluster/health || exit 1"]
      interval: 10s
      timeout: 10s
      retries: 3
    networks:
      - elastic
	
# Can't use logstash versions staring with 7.13 without a license (7.12 worked)
  logstash:
    image: docker.elastic.co/logstash/logstash-oss:7.12.1-amd64
    container_name: logstash
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf
    ports:
      - "5011:5011"
      - "5012:5012"
      - "5013:5013"
    networks:
      - elastic
    depends_on:
      - odfe-node1	
	  
  kibana:
    image: amazon/opendistro-for-elasticsearch-kibana:1.13.2
    container_name: odfe-kibana
    ports:
      - 5601:5601
    expose:
      - "5601"
    environment:
      ELASTICSEARCH_URL: https://odfe-node1:9200
      ELASTICSEARCH_HOSTS: https://odfe-node1:9200
    networks:
      - elastic

  nginx:
    image: nginx
    container_name: nginx-test
    volumes:
      - ./frontend/nginx/htpasswd:/etc/nginx/htpasswd.users
      - ./frontend/nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./frontend/ssl/awselk.crt:/etc/nginx/server.crt
      - ./frontend/ssl/awselk.key:/etc/nginx/server.key
      - ./frontend/ssl/ca.crt:/etc/ca.crt
    ports:
      - 80:80
      - 443:443
    networks:
      - elastic

networks:
  odfe-net:
  
volumes:
  odfe-data1:
```

> **Note:** You will need to create a sub directory "frontend" and under that directory you will need to create two more directories - "nginx" and "ssl". This is where the configuration files for the NGINX frontend and ssl configuration will be placed.

### Create nginx.conf File
**Step 7:** Time to create the nginx.conf file for the NGINX front end. Below is the contents of this file:

```console
worker_processes 1;

events { worker_connections 4096; }

http {
    
	sendfile on;
	
	upstream docker-kibana {
		server kibana:5601;
	}
	
	server {
		listen 80;
		server_name awselk.jkirn.com;
		
		location / {
			rewrite ^ https://$host$request_uri? permanent;
		}
	}
	
	server {
		listen 443 ssl;
		server_name awselk.jkirn.com;
    
#		auth_basic "Restricted Access";
#		auth_basic_user_file /etc/nginx/htpasswd.users;

		ssl_certificate 	/etc/nginx/server.crt;
		ssl_certificate_key	/etc/nginx/server.key;
		
		# Recommendations
		ssl_protocols TLSv1.1 TLSv1.2;
		ssl_ciphers 'EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH';
		ssl_prefer_server_ciphers on;
		ssl_session_cache shared:SSL:10m;
		
		# disable any limits to avoid HTTP 413 for large image upooads
		client_max_body_size 0;
		
		# required to avoid HTTP 411
		chunked_transfer_encoding on;
          
		location / {
			proxy_pass 				http://docker-kibana;
			proxy_set_header Host 			$http_host;
			proxy_set_header X-Real-IP 		$remote_addr;
			proxy_set_header X-Forwarded-For 	$proxy_add_x_forwarded_for;
			proxy_set_header X-Forwarded-Proto	$scheme;
			proxy_read_timeout			900;

		}

	}
}

```

### Create logstsh.conf File
**Step 8:** Time to create the logstash.conf file for the logstash collector. Below is the contents of this file:
```console
# Input
input {
    http {
      port => 5011
      codec => "line"
    }
    udp {
        port => 5012
        codec => "json"
    }
    tcp {
    	port => 5013
    	codec => "json_lines"
    }
}

# ---------------------------
#
# Filter stuff would go here
#
# ---------------------------

# Output goes to Elasticsearch
output {
    elasticsearch {
        hosts => ["https://odfe-node1:9200"]
        ssl => true
        ssl_certificate_verification => false
        user => "logstash"
        password => "logstash"
        ilm_enabled => false
        index => "logstash"
    }
    stdout {
    }
}
```

### Get CA Signed Server certificate info for SSL install
**Step 9:** Follow the steps documented at my github repository [Private-Certificate-Authority](https://github.com/jjkirn/Private-Certificate-Authority) to create your own CA and generate the target server certificate files for this server:
 - awselk.crt
 - awselk.key

That same document explains how to transfer these files from the CA server to this target (sysmon) server.

The above target sever ca files should be moved  the "ssl"  directory under the "frontend" directory.

### Launch the docker-compose Application
**Step 10:** In a terminal window where you created your docker-compose.yml file run the following command:
```console
docker-compose up
```
You should start seeing the starup messages.

### Sign in to Kibana
**Step 11:** Open up your browser, and go to the address that you assigned to your Kibana instance in the Nginx configuration. You should be directed to the welcome page where you enter the username and password that you set up for Kibana. The default username:password is admin:admin.
![Login Page](images/login.png)

Enter the user name and password and you should be rewarded with the Welcome page:
![Discover page](images/welcome.png)

Refer to the Open Distro for Elasticsearch Documentation reference link below (#9) for further details about Elasticsearch.

### References
1. [Ubuntu 20.04 (Focal Fossa) ISO Images](https://releases.ubuntu.com/focal/)
2. [Open Distro for Elasticsearch ISO Images](https://opendistro.github.io/for-elasticsearch/)
3. [logstash-oss:7.12.1-amd64 image](https://www.docker.elastic.co/r/logstash/logstash-oss)
4. [Preparing a Private Certificate Chain](https://github.com/jjkirn/Private-Certificate-Authority)
5. [Trying out Open Distro for Elasticsearch with Logstash](https://sjoerdsmink.medium.com/trying-out-open-distro-for-elasticsearch-with-logstash-8cc8be3e1e84)
6. [Getting started with Logstash](https://opensource.com/article/17/10/logstash-fundamentals)
7. [Install Docker Engine on Ubuntu (20.04)](https://docs.docker.com/engine/install/ubuntu/)
8. [Install Docker-Compose on Ubuntu (20.04)](https://docs.docker.com/compose/install/)
9. [Open Distro for Elasticsearch Documentation](https://opendistro.github.io/for-elasticsearch-docs/)



End of Document

