# Load Balancer Solution With Apache

In our set up in Project-7 we had 3 Web Servers and each of them had its own public IP address and public DNS name. A client has to access them by using different URLs, which is not a nice user experience to remember addresses/names of even 3 server, let alone millions of Google servers.

In order to hide all this complexity and to have a single point of access with a single public IP address/name, a Load Balancer can be used. A Load Balancer (LB) distributes clients’ requests among underlying Web Servers and makes sure that the load is distributed in an optimal way.

Let us take a look at the updated solution architecture with an LB added on top of Web Servers

https://professional-pbl.darey.io/en/latest/_images/Tooling-Website-Infrastructure-wLB.png

In this project we add a Load Balancer to disctribute traffic between Web Servers and allow users to access our website using a single URL.

### Task ###
Deploy and configure an Apache Load Balancer for Tooling Website solution on a separate Ubuntu EC2 intance. Make sure that users can be served by Web servers through the Load Balancer.

To simplify, let us implement this solution with 2 Web Servers, the approach will be the same for 3 and more Web Servers.

### Prerequisites

Two RHEL8 Web Servers
One MySQL DB Server (based on Ubuntu 20.04)
One RHEL8 NFS server


https://professional-pbl.darey.io/en/latest/_images/prerequisites-project8.png


### Configure Apache As A Load Balancer

1. Create an Ubuntu Server 20.04 EC2 instance and name it Project-8-apache-lb, so your EC2 list will look like this:

![alt text](https://github.com/olateekay/Loadbalancer-solution-with-Apache/blob/main/Images/Image1.png)

2. Open TCP port 80 on Project-8-apache-lb by creating an Inbound Rule in Security Group.
3. Install Apache Load Balancer on Project-8-apache-lb server and configure it to point traffic coming to LB to both Web Servers:

```
#Install apache2
sudo apt update
sudo apt install apache2 -y
sudo apt-get install libxml2-dev

#Enable following modules:
sudo a2enmod rewrite
sudo a2enmod proxy
sudo a2enmod proxy_balancer
sudo a2enmod proxy_http
sudo a2enmod headers
sudo a2enmod lbmethod_bytraffic

#Restart apache2 service
sudo systemctl restart apache2

```


4. Make sure apache2 is up and running

```
sudo systemctl status apache2

```

5. Configure load balancing

```
sudo vi /etc/apache2/sites-available/000-default.conf

#Add this configuration into this section <VirtualHost *:80>  </VirtualHost>

<Proxy "balancer://mycluster">
               BalancerMember http://<WebServer1-Private-IP-Address>:80 loadfactor=5 timeout=1
               BalancerMember http://<WebServer2-Private-IP-Address>:80 loadfactor=5 timeout=1
               ProxySet lbmethod=bytraffic
               # ProxySet lbmethod=byrequests
        </Proxy>

        ProxyPreserveHost On
        ProxyPass / balancer://mycluster/
        ProxyPassReverse / balancer://mycluster/

#Restart apache server

sudo systemctl restart apache2

```


*NB* : `bytraffic`  balancing method will distribute incoming load between your Web Servers according to current traffic load.


6. Verify that our configuration works - try to access your LB’s public IP address or Public DNS name from your browser:
http://<Load-Balancer-Public-IP-Address-or-Public-DNS-Name>/index.php

![alt text](https://github.com/olateekay/Loadbalancer-solution-with-Apache/blob/main/Images/Image2.png)

7. Unmount `/var/log/httpd/` from your Web Servers to the NFS server and make sure that each Web Server has its own log directory.

`sudo umount -t nfs ,nosuid <NFS-Server-Private-IP>:/mnt/apps /var/www/`

8. Try to refresh your browser page 

`http://<Load-Balancer-Public-IP-Address-or-Public-DNS-Name>/index.php` 

several times and make sure that both servers receive HTTP GET requests from your LB - new records must appear in each server’s log file. The number of requests to each server will be approximately the same since we set loadfactor to the same value for both servers - it means that traffic will be disctributed evenly between them.


![alt text](https://github.com/olateekay/Loadbalancer-solution-with-Apache/blob/main/Images/Image3.png)

### Configure Local DNS Names Resolution

Sometimes it is tedious to remember and switch between IP addresses, especially if you have a lot of servers under your management. What we can do, is to configure local domain name resolution. The easiest way is to use `/etc/hosts` file, although this approach is not very scalable, but it is very easy to configure and shows the concept well. So let us configure IP address to domain name mapping for our LB.

```
#Open this file on your LB server

sudo vi /etc/hosts

#Add 2 records into this file with Local IP address and arbitrary name for both of your Web Servers

<WebServer1-Private-IP-Address> Web1
<WebServer2-Private-IP-Address> Web2

```

Now you can update your LB config file with those names instead of IP addresses.


```
BalancerMember http://Web1:80 loadfactor=5 timeout=1
BalancerMember http://Web2:80 loadfactor=5 timeout=1

```

You can try to curl your Web Servers from LB locally `curl http://Web1` or `curl http://Web2` 


```
Output:

<!DOCTYPE html>
<html>

<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link rel="stylesheet" type="text/css" href="tooling_stylesheets.css">
  <script src="script.js"></script>
  <title> PROPITIX TOOLING</title>
</head>

<body>
  <div class="header">
  </div>

  <div class="box">
    <a href="https://prometheus.infra.zooto.io/" target="_blank">
      <img src="img/prometheus.png" alt="Snow" width="400" height="150">
    </a>
  </div>
  <div class="box">
    <a href="https://k8s-metrics.infra.zooto.io/" target="_blank">
      <img src="img/kubernetes.png" alt="Snow" width="400" height="120">
    </a>
  </div>
  <div class="box">
    <a href="https://kibana.infra.zooto.io/" target="_blank">
      <img src="img/kibana.png" alt="Snow" width="400" height="10
0">
    </a>
  </div>
  </div>
  <div class="container">
    <div class="box">
      <a href="https://artifactory.infra.zooto.io/" target="_blank">
        <img src="img/jfrog.png" alt="snow" width="400" height="100
">
      </a>
    </div>
  </div>
  </div>
  </section>
</body>

</html>


```


You have just implemented a Load balancing Web Solution.

