# GitLab on a Raspberry Pi Cluster

How to get GitLab running on a Raspberry Pi cluster.

[Gitlab have instructions](https://about.gitlab.com/installation/#raspberry-pi-2) on how to install GitLab on Raspberry Pi 2 units - presumably the same steps will work on Raspberry Pi 3 units as well.

However, performance is terrible and it's basically unusable on the latest version of Raspian (Debian Stretch at the time of writing).

If you have 2 RPi units, you can create a cluster which is perfectly suitable for personal development purposes - response times are ~3 seconds, and you will not run into memory issues.

These instructions are based on [the guide found here](https://evil-zebra.com/run-gitlab-ce-on-a-raspberry-pi-3-cluster/). I have created this file in case the original steps become unavailable.

## Step 1: Set up the RPi units
 
Install Raspian (Debian Stretch at the time of writing) on both units.

```bash
sudo apt-get update
sudo apt-get upgrade
sudo raspi-config
```

Now set the hostname for one RPi as ‘gitlab’, and the other to ‘gitlab-core’.
‘gitlab-core’ should be the RPi with the biggest SD card, since this is where the data is held.

**Use raspi-config to set the hostname***

On both RPi units, make the following changes via rasp-config:
 
* Set SSH to be on for both
* Set WiFi details - type in the access point you want to connect to
 
If you're using RPi 2 B+ units, connect a WiFi USB dongle. This will automatically connect to your WiFi network when you reboot after installing Zram.
 
## Step 2: Install ZRAM
 
Instead of increasing swap space, the general consensus is to use [ZRAM](https://github.com/novaspirit/rpi_zram/).

To do this, download the script and copy to /usr/bin/ folder

```bash
sudo wget -O /usr/bin/zram.sh https://raw.githubusercontent.com/novaspirit/rpi_zram/master/zram.sh
```

Once the script is downloaded we will need to change the permissions so it can be an executable.

```bash
sudo chmod +x /usr/bin/zram.sh
```

Edit /etc/rc.local file to run script on boot:

```bash
sudo nano /etc/rc.local
```

Add this line before exit 0 (at the end):

```bash
$ /usr/bin/zram.sh &
``` 
 
Now reboot the Pi. If you're interested in seeing the difference, type 'free' before and after the reboot to see how much swap space you now have.

## Step 3: Set up the 'gitlab' Pi
  
### Install nginx

You will need to manually install a web-server, GitLab uses nginx by default and is lightweight so it is a logical choice.
 
```bash
sudo apt-get install nginx
 ```
 
### Configure nginx

You will need to tell nginx to communicate with your other Pi for handling requests. To do this you need to change some configuration in nginx. Copy [this config text](https://gitlab.com/gitlab-org/gitlab-recipes/blob/master/web-server/nginx/gitlab-omnibus-nginx.conf) into /etc/nginx/sites-enabled/gitlab.

```bash
sudo nano /etc/nginx/sites-enabled/gitlab
```

Insert the nginx config text and make a couple edits to it.

```bash
# nginx can not use a unix sock
upstream gitlab-workhorse {
    server 192.168.1.1:8181;
}
```
**This should match /etc/gitlab/gitlab.rb external_url**

```bash
server_name gitlab.local;
```

Now you need to delete the default nginx configuration.
 
```
sudo rm /etc/nginx/sites-enabled/default
```

### Setup Static Ethernet
You will be connecting the two Pi’s via an Ethernet cable, you do not have to plugin the cable yet but you can still setup the Pi to use a static IP for Ethernet.
 
Open in text editor of your choice:
```bash
sudo nano /etc/dhcpcd.conf
```

Add and save the following to the end or beginning of the file.

```ini
# static ethernet connection
interface eth0
static ip_address=192.168.1.2/24
static routers=192.168.1.1
static domain_name_servers=192.168.1.1
 
interface wlan0
static ip_address=192.168.1.2/24
static routers=192.168.1.1
static domain_name_servers=192.168.1.1
```

### Install GitLab CE
Okay, now you can finally install GitLab CE. Once again there are perfectly clear instructions on the official GitLab CE page (https://about.gitlab.com/installation/#raspberry-pi-2), please follow steps 1 and 2. You will not need to install postfix, unless you wish to play around with email notifications.
 
You need to setup GitLab to run only the required services. You can either uncomment and edit each of the values below in the default config file or you can just copy this config into the beginning of the file.
 
```ini
# open in text editor of your choice
sudo nano /etc/gitlab/gitlab.rb
 
# must be the same as other Pi
external_url "http://gitlab.local"
# only main GitLab server should handle migrations
gitlab_rails['auto_migrate'] = false
# enable postgresql
postgresql['enable'] = true
postgresql['listen_address'] = '192.168.1.2'
# allow the other Pi to connect to postgresql
postgresql['custom_pg_hba_entries'] = {
    TCP_HOST: [{
        type: 'host',
        database: 'all',
        user: 'gitlab',
        cidr: '192.168.1.0/24',
        method: 'trust',
        option: ''
    }]
}
# enable Redis
redis_master_role['enable'] = true
# setup Redis TCP support
redis['bind'] = '192.168.1.2'
redis['port'] = 6379
redis['password'] = 'your-redis-password'
```

Save and reconfigure:

```bash
sudo gitlab-ctl reconfigure
```
 
At this point you can shutdown the Pi and move on to setting up the next Pi.
 
## Step 4: Setup main GitLab Pi (Gitlab-core)
  
### Setup Static Ethernet
This Pi will also need a static Ethernet address, same process as before but the IP address is different.
 
```bash
sudo nano /etc/dhcpcd.conf
```

Add and save the following to the end or beginning of the file.

```ini
# static ethernet connection
interface eth0
static ip_address=192.168.1.1/24
static routers=192.168.1.2
static domain_name_servers=192.168.1.2
 
interface wlan0
static ip_address=192.168.1.1/24
static routers=192.168.1.2
static domain_name_servers=192.168.1.2
 ```
 
### Connect Devices

It is time to connect the two Raspberry Pi’s to each other so that your main GitLab Pi can communicate with the web-server Pi. Connect your Ethernet cable and then do a reboot to restart services. At this point you should plugin your web-server Pi, if it is not already on. If it is I would recommend rebooting that device as well.
 
### Restart the device

```bash
sudo reboot
```
 
### Install & Configure GitLab CE
Follow the previous section on installing GitLab. Once again, we are only doing steps 1 and 2.
 
Next you will need to configure some addresses and disable some services. You will be editing /etc/gitlab/gitlab.rb again:
 
```ini
# setup an easy to remember and easy to type url
external_url "http://gitlab.local "
# GitLab will need to trust our web-server
gitlab_rails['trusted_proxies'] = ['192.168.1.2']
web_server['external_users'] = ['www-data']
# our other pi will run database
gitlab_rails['db_host'] = '192.168.1.2'
# our other pi will also run Redis
gitlab_rails['redis_host'] = '192.168.1.2'
# enter a random password for Redis, we will set the password later
gitlab_rails['redis_password'] = 'your-redis-password'
# Apache does not support UNIX socket so we must enable TCP, our web-server will use this service
gitlab_workhorse['listen_network'] = 'tcp'
gitlab_workhorse['listen_addr'] = '192.168.1.1:8181'
# Now lets disable some services that our other Pi will handle
postgresql['enable'] = false
redis['enable'] = false
nginx['enable'] = false
```

Save that file and reconfigure GitLab. Make sure both Pi’s are running before reconfigure or you will get an error.
 
```bash
sudo gitlab-ctl reconfigure
```
 
## Step 5: Start up GitLab CE

Go to http://gitlab.local in a browser.



