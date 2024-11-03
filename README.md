# Pi5-Ubuntu-Server

## Building a Raspberry Pi 5 Ubuntu Server
Using a Raspberry Pi 5 (8GB RAM) single board computer, I will configure a server on the home lab subnet created thus far. On this server I plan to set up network services for my local network. The services will provide monitoring and data collection of my lab network, devices, and the server device itself. 
> **Note:** I am somewhat regretful about choosing the Pi 5 for this project. I have considered using another mini PC with an x86 architecture instead of the ARM64 that the Pi offers. While the ARM64 offers many native application solutions from official distributions to community built projects, the x86 architecture is superior for compatibility and performance, including the ability to run enterprise level applications.

Mini PCs are also expandable in terms of memory and storage, while having comparable performance at an arguably similar price point. Although they use slightly more power, I feel it is worth the gain in performance. I may convert this work over to an x86 setup later, but for the time being this project will offer valuable experience.

> **Note:** In a later project, I will discuss the use of security configurations to harden these services. Including a reverse proxy service (for encryption and SSL certificate management)

## Hardware Assembly
The Pi 5 is a single board computer with better specifications than previous models. Some important features include:
- 2.4GHz quad-core 64-bit Arm Cortex-A76 CPU
- 8GB RAM LPDDR4X-4267 SDRAM
- 2 × USB 3.0 ports
- 2 × USB 2.0 ports
- Gigabit Ethernet, with PoE+ support (requires additional HAT)

In this section I will simply place the Pi 5 board into a case to protect it. The case also includes a fan for better cooling under workload. The Pi 5 board and case are seen below in the figure below. The fan unit is built into the casing mold and can be identified by the wiring. All black portions are made of aluminum, which serves as a great heatsink.

<p align="center">
  <br/>
  <img src="https://imgur.com/7o78bjB.png" height="80%" width="80%" alt="Table"/><br /><br />
</p

In the following image, the Pi 5 board has been placed in the chassis casing and the fan wiring has been plugged in to the appropriate port. The wiring has been carefully placed in a channel that guides the cable from where it exits the fan at A, then follows the wiring channel along B, and finally exits at C, to plug into the fan port on the Pi board.

<p align="center">
  <br/>
  <img src="https://imgur.com/BFUJWKt.png" height="80%" width="80%" alt="Table"/><br /><br />
</p>

The two images above show the bottom of the casing (colored red) and how it leaves access to the SD card slot exposed. This can be covered with a small plastic lid and affixed with a screw. Finally, the last image shows the completed case construction, which can be opened for access by the two black screws on top.


# Installing Ubuntu Server
The goal of this phase of the project is to install Ubuntu Server OS onto a Raspberry Pi 5 hardware device and use that server to run monitoring services that will provide information about our network, devices (if they are configured to be scraped for data), security, configurations, resource utilization, docker container data, and our server itself. It is a large project and will be an ongoing process, but I plan to get the basic foundations completed.

## Flashing Ubuntu ARM64 image to SD card
- Download the Pi imager software here (be sure to choose your correct OS)
- Insert the SD card into a USB adapter and plug into USB port
- Install and open the Pi imager software
  - Choose the correct pi hardware (Pi 5 for our scenario)
  - Choose the OS to flash (Ubuntu Server 64bit)
  - Edit OS settings
    - Set username and password
    - Enable SSH and choose password authentication
  - Set the correct target storage to flash the image to (do not choose the wrong target, it can result in serious data loss)
  - Flash the image
  - After it is completed
    - Unplug the SD card
    - Place the SD card into the Pi
    - Power on the device and connect ethernet cabling

## Connecting to the Pi with SSH
This is a headless configuration (there is no display connected to the Pi server. I will connect to it from a separate computer on the same subnet.
- First I need the IP address of the Pi that was just connected to the network
  - It should be given an IP address by the DHCP server for this subnet
  - There are many ways to check (nmap, ARP, Advanced IP scanner)
  - I chose to sign into my firewall GUI and check the DHCP leases because other methods did not give a name to the device
- Use Windows Terminal or download PuTTY here (or SolarPuTTY, etc.)
- The command `ssh <username>@<IP address>` will attempt to connect to the device
- This will bring up a warning that the device is unknown and the ssh key is not recognized, ‘would you like to store it’?
  - Choose yes and the connection is made, and the encrypted key is stored for later authentication

In the figure below, I can see the SSH connection is made to the headless server. Note the IP address has been given by the DHCP server for the lab subnet. Which is subnet 192.168.x.x /24. I will change this IP in the next steps.


<p align="center">
  <br/>
  <img src="https://imgur.com/oEguRC8.png" height="80%" width="80%" alt="Table"/><br /><br />
</p>



### Basic setup for the Linux server
The following sections will have a lot of lines of code to enter. It is entirely a command line interface on the server and may be frustrating at first, but practice will improve our skills. The basic tasks I will perform are to:
- Check the date and time on the device
- Update the system (it will not update if the date/time are incorrect)
- Change to a static IP address
- Install Docker
- Create a Portainer docker container

### Check the date and time on the device
I had issues connecting to the time servers and needed to set the time manually.
- Enter `date`
  - Note it is incorrect, you cannot update if the date/time are wrong
- `sudo systemctl status systemd-timesyncd`
- `sudo timedatectl set-ntp false`
- `sudo timedatectl set-time` '2024-10-30 14:30:00'  # Adjust to current time

### Update the system
- `sudo apt update && sudo apt upgrade -y`

### Change the device to a static IP
- `sudo systemctl status systemd-networkd`
- `sudo nano /etc/netplan/50-cloud-init.yaml`
  - This brings up a configuration file in the nano editor, the following is the configuration file contents to enter and save.
```
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: no
      addresses:
        - 192.168.x.x/24  # Your desired static IP address
      routes:
        - to: 0.0.0.0/0      # Default route
          via: 192.168.x.x   # Your gateway IP address
      nameservers:
        addresses:
          - 8.8.8.8          # Primary DNS (example, use what you wish)
          - 8.8.4.4          # Secondary DNS (example, use what you wish)
```

- `sudo netplan apply`
- `ip a`

# Docker and Containers

## Install Docker
- `sudo apt update`
- `sudo apt install -y ca-certificates curl gnupg`
- `sudo install -m 0755 -d /etc/apt/keyrings`
`curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg`
- `echo "deb [arch=arm64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null`
- `sudo apt update`
- `sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin`
- `sudo systemctl start docker`
- `sudo systemctl enable docker`
- `docker –version`
- `sudo usermod -aG docker $USER`

## Create Portainer docker container
- `docker pull portainer/portainer-ce:linux-arm64`
- `docker volume create portainer_data`
- `docker run -d -p 8000:8000 -p 9443:9443 --name portainer \ -v /var/run/docker.sock:/var/run/docker.sock \ -v portainer_data:/data \ portainer/portainer-ce:linux-arm64`
- Visit GUI `https://pi_address:9443`
- Setup username and password


## Downloading and editing configuration files
There are three configuration files I will be concerned with during this section. These will not be written from scratch. I will be using a basic template, but I will need to change a few things to make sure the services communicate and function properly. Each file can be downloaded from GitHub user James Turland at Jims Garage. The three files are:

- A docker compose file that is similar to a script and allows you to build a configure multiple containers simultaneously.
- A Prometheus configuration file to specify targets you wish to scrape data from, as well as details regarding which data, how often, etc.
- A Telegraf configuration file that will scrape data from Docker regarding the containers and their related data (number of containers running, resource usage, etc.) 

These files are configured for the specifics of James’ network, and I will need to change many things to make them work for my scenario. This is a trial-and-error process that requires a lot of troubleshooting to achieve functionality. I will cover some of the basic changes that need to be made but I cannot include every detail in this documentation.
- Downloading a file transfer application like WinSCP for secure file transfer may make things easier to understand the Linux file structure. You can also use the `nano` text editor inside Ubuntu to copy and paste the raw file data and save it that way. This will require you to create directories and files with the CLI.
I constructed the following file structure in Linux to prepare before moving the configuration files.

- home/blaine/
  - docker/
    - grafana-monitoring/
      - grafana
      - graphite
      - influxDB
      - loki
      - prometheus
        - config/
          - prometheus.yml
      - promtail
      - telegraf/
        - telegraf.conf
  - docker-compose/
    - grafana-monitoring/
      - docker-compose.yaml

This was not explained by the user that provided the docker-compose file. It took some time and troubleshooting for me to understand why this structure was needed. 

## Building the containers with `docker compose`
I then went to the /docker-compose/grafana-monitoring/ directory because that is where our compose file is placed. Then I ran the compose file  to build the containers in detached mode using the command.
- `sudo docker compose up -d`

This will build the containers and place all other related files into the structure I have built.


<p align="center">
  <br/>
  <img src="https://imgur.com/OOyEGh3.png" height="80%" width="80%" alt="Table"/><br /><br />
</p>



Then I navigated to the portainer GUI from earlier to review the containers I just created, check their logs, and ensure that they were functioning. They were running but not communicating properly. By troubleshooting I realized that I needed to change some of the following items:

- `docker-compose.yaml`
- remove `Version: 3`
- change each volume path from `home/ubuntu`, to `home/blaine`
- find the GID of Docker in Ubuntu and change the line `user:`
- ensure line `network:` read `grafana-montoring: driver: bridge`

Later I will discuss the changes to other configuration files as they become relevant. I confirmed that each container was functional by reviewing the log output for each in Portainer. I then went to each container’s webGUI with the IPaddress:port combination and set up usernames and passwords for each one.


Navigating to the Portainer webGUI I can see that all the containers are running as they should be, this is where I can investigate the logs for each container for more details as well. This can be done in the server CLI, but GUIs provides a quick method that is easier to interpret for many users.


<p align="center">
  <br/>
  <img src="https://imgur.com/iVTv9Mv.png" height="80%" width="80%" alt="Table"/><br /><br />
</p>


## Grafana
This will serve as our main visualization application and the location for building our dashboards. It does not do any collection or scraping of data. Grafana serves as a platform to push data and metrics into and a tool to visualize it in an organized format. An image of a Grafana dashboard will be examined later to demonstrate its visualization capabilities.

## InfluxDB
InfluxDB serves as a time series database (TSDB). It manages large amounts of data and metrics. The information that is held in InfluxDB will later be acted upon by Grafana to build the dashboard interfaces. InfluxDB does not collect the data itself. That is what Telegraf (as well as Prometheus) will be used to do.
- Go to the InfluxDB webGUI addres
- Create name and password
- Add an organization name and bucket name
- Create API token (copy and paste into notepad, the key can only be viewed once)

In the following image I can see that InfluxDB can be used to visualize some data, but that is not its main purpose, and it is weaker at this task than Grafana



<p align="center">
  <br/>
  <img src="https://imgur.com/pnIRg2j.png" height="80%" width="80%" alt="Table"/><br /><br />
</p>



## Telegraf configuration file
As I recently mentioned, Telegraf will scrape data, pass it on to InfluxDB, and then Grafana will act upon the data stored in InfluxDB to create visually appealing and informative dashboards of disparate and scattered data.
- Go to `telegraf.conf` and add (this information is in the webGUI):
```
[[outputs.influxdb_v2]]
## The URLs of the influxDB cluster nodes
 ##
  ## Multiple URLs can be specified for a single cluster, one ONE of the
  ## urls will be written to each interval.
  ##   ex: ["https://us-west-2-1.aws.cloud2.influxdata.com"]
  #urls = ["http://influxdb:8086"]

  ## API token for authentication
  token = "this is where the generated token goes”

  ## Organization is the name of the organization you wish to write out to.
  This must exist.
  organization = "home"

  ## Destination bucket to write into.
  bucket = "homelab"
```
- Save
- Restart telegraf docker container
`sudo docker restart telegraf`


Viewing the custom dashboard in Grafana
The figure shown below is the custom dashboard creating by using Grafana to visualize Docker data from InfluxDB after it has been scraped and pushed there by Telegraf. It may seem complicated, and it is, but it follows the path of:
- Docker running containers for services
- Telgraf scraping the Docker container data, as well as server metrics, and pushing it to InfluxDB
- InfluxDB managing and storing this data
- Grafana acting on this data and organizing it into appealing charts


<p align="center">
  <br/>
  <img src="https://imgur.com/a13v0fb.png" height="80%" width="80%" alt="Table"/><br /><br />
</p>


This is all very customizable and this documentation only scratches the surface of what is possible. I plan to coordinate the other services to organize other data like SIEM, firewall logging, network performance, and configuration changes. Two containers are not running (Loki and Promtail), and I know this from the logs available in Portainer which read:
- Unable to parse config: `/etc/promtail/promtail-config.yml` does not exist, set config.file for custom config path
- Failed parsing config: `/etc/loki/loki-config.yml` does not exist, set config.file for custom config path
These containers need further configuration work, which I plan to perform in the future.

References
- https://github.com/grafana
- https://github.com/JamesTurland/JimsGarage/tree/main/Grafana-Monitoring
- https://github.com/influxdata/telegraf/releases
- https://github.com/prometheus/prometheus/releases
- https://hub.docker.com/layers/grafana/promtail/master-b652f0a-arm64/images/sha256-3ee38cc0306e6d22e42d6af238236289a44229a906efbec2da4285e0a7e984d7?context=explore 



