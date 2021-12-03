---
---
# Installing Wireguard on Docker

## Setting up the VM

I first created a DigitalOcean droplet running Ubuntu 20.04 to host this project, and installed Docker on it. First, I installed the packages needed to verify a certificate with ```sudo apt install apt-transport-https ca-certificates curl software-properties-common -y```, then installed the Docker certificate using ```curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -```. 

After this, I then obtained the docker source for an x86-64 CPU, since that is what my droplet is running on, using the command ```sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu$(lsb_release -cs)stable" && apt-cache policy docker-ce```, then installed docker with the command ```sudo apt install docker-ce -y```. 

After verifying that docker was in fact installed by running ```docker --help``` and getting a valid help menu, I installed docker-compose from GitHub using ```sudo curl -L "https://github.com/docker/compose/releases/download/1.27.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose```, and enabled the command with ```sudo chmod +x /usr/local/bin/docker-compose```. After

## Installing Wireguard

I first created the `~/wireguard/` and `~/wireguard/config/` directories, to be used by the docker image, and then created a `docker-compose.yml` file with the following contents:
```yml
version: '3.8'
services:
  wireguard:
    container_name: wireguard
    image: linuxserver/wireguard
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Chicago
      - SERVERURL=143.198.147.233
      - SERVERPORT=51820
      - PEERS=pc1,pc2,phone1
      - PEERDNS=auto
      - INTERNAL_SUBNET=10.0.0.0
    ports:
      - 51820:51820/udp
    volumes:
      - type: bind
        source: ./config/
        target: /config/
      - type: bind
        source: /lib/modules
        target: /lib/modules
    restart: always
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
```
I then ran ```docker-compose up -d``` to start the Wireguard docker image.

## Testing

I installed the Wireguard mobile app on my phone and scanned the generated QR code from running ```docker-compose logs -f wireguard``` to enable the VPN. Before, my IP was as shown

![Mobile IP Before](/images/wgmobilebefore.png)

And afterwards, with the VPN active, my IP changed to the following:

![Mobile IP After](/images/wgmobileafter.png)

I then copied the config file from ```~/wireguard/config/peer_pc1/peer_pc1.conf``` to the Wireguard desktop app to set up the VPN on my laptop, and connected to it as shown below:

![Laptop running Wireguard](/images/wglaptop.png)
