## Jenkins Setup

- [Prerequisites](#prerequisites)
- [Start Jenkins](#start-jenkins)
- [Post-installation setup wizard](#post-installation-setup-wizard)
- [Unlocking Jenkins](#unlocking-jenkins)
- [Jenkins Service](#jenkins-service) 
- [plugins selection and installation](#plugins-selection-and-installation)
- [Admin Account Creation](#admin-account-creation)
- [Change Default Port](#change-default-port)
- [SSL and Nginx](#ssl-and-nginx) 


Jenkins is an open source continuous integration/continuous delivery and deployment (CI/CD) automation software DevOps tool written in the Java programming language. It is used to implement CI/CD workflows, called pipelines.

### Prerequisites

| Minimum hardware requirements | Recommended hardware configuration |
| ----- | ----------- |
|1 GB of RAM | 8 GB+ of RAM|
|10 GB of drive space | 100 GB+ of drive space|

You need to add the Jenkins repository from the Jenkins website to the package manager first. It can be installed from the redhat-stable yum repository.

```bash
$ sudo wget -O /etc/yum.repos.d/jenkins.repo \
    https://pkg.jenkins.io/redhat-stable/jenkins.repo

$ sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key

$ sudo dnf upgrade
# Add required dependencies for the jenkins package
$ sudo dnf install java-11-openjdk
$ sudo dnf install jenkins
$ sudo systemctl daemon-reload
``` 
## Start Jenkins

You can enable the Jenkins service to start at boot with the command:
```bash 
$ sudo systemctl enable jenkins 
```

You can start the Jenkins service with the command:
```bash
$ sudo systemctl start jenkins
```

You can check the status of the Jenkins service using the command:

```bash 
$ sudo systemctl status jenkins.service 

jenkins.service - Jenkins Continuous Integration Server
     Loaded: loaded (/usr/lib/systemd/system/jenkins.service; enabled; vendor preset: disabled)
     Active: active (running) since Thu 2022-09-08 15:39:55 IST; 6min ago
   Main PID: 23747 (java)
      Tasks: 49 (limit: 9485)
     Memory: 1.5G
        CPU: 55.470s
     CGroup: /system.slice/jenkins.service
             └─ 23747 /usr/bin/java -Djava.awt.headless=true -jar /usr/share/java/jenkins.war --webroot=/var/cache/jenkins/war --httpPort=8080

```
If everything has been set up correctly, you should see an output like this:
```
Loaded: loaded (/lib/systemd/system/jenkins.service; enabled; vendor preset: enabled)
Active: active (running) since Tue 2018-11-13 16:19:01 +03; 4min 57s ago ...
```
If you have a firewall installed, you must add Jenkins as an exception. You must change `<PORT>` in the script below to the port you want to use. Port 8080 is the most common.

```bash
firewall-cmd --add-port=<PORT>/tcp --permanent
firewall-cmd --reload
```

By using above commands we can start Jenkins
## Post-installation setup wizard

After downloading, installing and running Jenkins using one of the procedures above, the post-installation setup wizard begins.

This setup wizard takes you through a few quick "one-off" steps to unlock Jenkins, customize it with plugins and create the first administrator user through which you can continue accessing Jenkins.


## Unlocking Jenkins
  When you first access a new Jenkins instance, you are asked to unlock it using an automatically-generated password.

1. Browse to `http://<IP>:<PORT>` (or whichever port you configured for Jenkins when installing it) and wait until the Unlock Jenkins page appears.

![Jenkin's admin password page](./assets/images/init-password.png)


**Note:**

- The command: `sudo cat /var/lib/jenkins/secrets/initialAdminPassword` will print the password at console.
-  On the Unlock Jenkins page, paste this password into the Administrator password field and click Continue.


### Plugins selection and installation

Select plugins to install and install required plugins.
    
![Jenkin's plugin selection page](./assets/images/plugin-selection.png)

Click on select Plugins to install 
![Jenkin's plugin page](./assets/images/plugin-page.png)

Click on Install Button
![Jenkin's plugin installation page](./assets/images/plugin-installation-view.png)

### Admin Account Creation

Let us create `admin` account for Jenkins

![Jenkin's Admin Account creation page](./assets/images/create-admin-account.png)

> _NOTE: In the next page skip the Jenkins URL as of now._

click `Start using Jenkin` button.

![Jenkin's jenkins ready page](./assets/images/jenkins-ready.png)


### Change Default Port

To change port at jenkins service run this command
```bash
$ sudo systemctl edit jenkins.service
```

```conf
### Editing /etc/systemd/system/jenkins.service.d/override.conf
### Anything between here and the comment below will become the new contents of the file

[Service]
Environment="JENKINS_PORT=<PORT>"

### Lines below this comment will be discarded
### /usr/lib/systemd/system/jenkins.service
```

```bash
$ sudo systemctl restart jenkins.service

$ sudo firewall-cmd --add-port=<PORT>/tcp --permanent
success

$ sudo firewall-cmd --reload
success
```

Open the browser and check if port is updated


#### SSL and Nginx
There are two ways to get certs for SSL
- using cerbot
- manual downloading it as zip file


We are going to use nginx as a proxy server and the SSL are unziped in `/opt/jenkins_cert/`

Run the command to install nginx

```bash
$ sudo dnf install nginx
```
enable nginx 

```bash
$ sudo systemctl enable --now nginx.service
``` 
Check if nginx is running

![nginx-default-page](./assets/images/nginx-default-page.png)

backup original nginx conf

```bash
$ sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.orig
```
Replace the nginx configuration and edit _lines starting with comment `<--`_

```bash
$ vi /etc/nginx/nginx.conf
```
```bash
# For more information on configuration, see:
#   * Official English Documentation: http://nginx.org/en/docs/
#   * Official Russian Documentation: http://nginx.org/ru/docs/

user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 4096;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include /etc/nginx/conf.d/*.conf;

  upstream jenkins {
    keepalive 32; # keepalive connections
   server 127.0.0.1:<PORT>; # jenkins ip and port        <---- Change port number
 }

# Required for Jenkins websocket agents
map $http_upgrade $connection_upgrade {
  default upgrade;
  '' close;
}

server {

  server_name     xxxx.corp.mastechinfotrellis.com;    # <---------- change server name

  # this is the jenkins web root directory
  # (mentioned in the output of "systemctl cat jenkins")
  root            /var/lib/jenkins;

  access_log      /var/log/nginx/jenkins.access.log;
  error_log       /var/log/nginx/jenkins.error.log;

  # pass through headers from Jenkins that Nginx considers invalid
  ignore_invalid_headers off;

  location ~ "^/static/[0-9a-fA-F]{8}\/(.*)$" {
    # rewrite all static files into requests to the root
    # E.g /static/12345678/css/something.css will become /css/something.css
    rewrite "^/static/[0-9a-fA-F]{8}\/(.*)" /$1 last;
  }

  location /userContent {
    # have nginx handle all the static requests to userContent folder
    # note : This is the $JENKINS_HOME dir
    root /var/lib/jenkins/;
    if (!-f $request_filename){
      # this file does not exist, might be a directory or a /**view** url
      rewrite (.*) /$1 last;
      break;
    }
    sendfile on;
  }

  location / {
      sendfile off;
      proxy_pass         http://jenkins;
      proxy_redirect     default;
      proxy_http_version 1.1;

      # Required for Jenkins websocket agents
      proxy_set_header   Connection        $connection_upgrade;
      proxy_set_header   Upgrade           $http_upgrade;

      proxy_set_header   Host              $host;
      proxy_set_header   X-Real-IP         $remote_addr;
      proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
      proxy_set_header   X-Forwarded-Proto $scheme;
      proxy_max_temp_file_size 0;

      #this is the maximum upload size
      client_max_body_size       10m;
      client_body_buffer_size    128k;

      proxy_connect_timeout      90;
      proxy_send_timeout         90;
      proxy_read_timeout         90;
      proxy_buffering            off;
      proxy_request_buffering    off; # Required for HTTP CLI commands
      proxy_set_header Connection ""; # Clear for keepalive
  }
  listen [::]:443 ssl ipv6only=on;
  listen 443 ssl;
  ssl_certificate "/opt/jenkins_cert/certificate.crt";          # <----  change cert path
  ssl_certificate_key "/opt/jenkins_cert/private.key";          # <----  change private key path
  ssl_session_cache shared:SSL:1m;
  ssl_session_timeout  10m;
  ssl_ciphers PROFILE=SYSTEM;
  ssl_prefer_server_ciphers on;

}
  server {
      if ($host = xxxx.corp.mastechinfotrellis.com) {       # <------- change server name
          return 301 https://$host$request_uri;
      }


        listen       80 default_server;
        listen       [::]:80 ipv6only=on default_server;
        server_name  xxx.corp.mastechinfotrellis.com;       # <---- change server name
    return 404;
  }
}
```

restart nginx service

```bash
$ sudo systemctl restart nginx.service
```

Check if nginx and SSL is working in browser

![jenkins-default-Homepage](./assets/images/jenkins-default-home-page.png)


### Add Agent in jenkins

- Click **Manage Jenkins** -> **Manage Nodes and Clouds**
- Click on **New Node**
- Enter node name and select **Permanent node** option
- Click **Create** button

After node is created click on configure at left sidepanel
- Increase **number of executors** (note: initially the number of executors should not exceed cpu count). Play around the value to find the sweet spot of executors.
- Provide **remote root directory** path (this is where job data is stored)
- Under **Launch method** select **SSH**
  - provide hostname
  - click on add button and include the remote user credentials
  - select the **credentials** in the dropdown now
  - Host key validation should be **Known host file verification strategy**
- Click save button

To make the SSH connection between jenkins controller and agent node we have to add the ssh keys of agent in jenkins machine.

Since the jenkins is handled by `jenkins:jenkins` user and group inside `/var/lib/jenkins` we have to create `.ssh` folder in there

```bash
$ mkdir -p /var/lib/jenkins/.ssh
```
to find and add ssh keys of agent node
```bash
$ ssh-keyscan -H <agent-host-name> >> /var/lib/jenkins/.ssh/known_hosts
```
Change the .ssh folder permissions to jenkins user

```bash
$ sudo chown -R jenkins:jenkins /var/lib/jenkins/.ssh/
```

if everything is done you should see **logs** under  Click **Manage Jenkins** -> **Manage Nodes and Clouds** -> **agent-name**

![ssh-connected-agent](./assets/images/agent-connected-ssh.png)

