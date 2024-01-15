# How-I-fixed-my-lancahe-and-pihole-not-working-together

I hope my writup will help yourself if you want to deploy a similar setup like mine without bashing your head against the wall for 3 weeks. I hope that this is the right place for something like an guide / documentation. If there is a better place please let me know and I'll move it over there. If you got any ideas to optimize my setup Please let me know.

My Setup / Your Setup after following this writeup
=

**Synology NAS 923+ that runs**

- a Linux Mint VM that runs Lancache, Portainer Agent, Docker, Docker Compose

**Synology NAS 923+ that runs**

- Portainer, Docker

**Rasberry pi 4 with DietpiOS running**

- Docker, Docker Compose, and a modified Pihole .yml

Creating a VM on the Synology NAS to host lancache properly.
=

1. Install Virtual Machine Manager on your Synology Nas
2. Download a Linux .iso of your choice In my case I used Mint, so if you choose something different you should have the knowledge to install lancache yourself. Maybe you just want to use my writeup to figure out what went sideways with your pihole installation. Anyway after downloading the .iso of you choice transfer it to your Nas.
3. If you dont have any VM storage configured you need to configure it. \[Storage -\> add -\> select the storage of your choice\]
4. Create a VM `Virtual machiene manager  -> virtual machine -> Create` Select Linux, your configured storage, name your VM (In my case "LancacheVM), give it 2 cpu, `Memory as much as youre comfortable with` (I used 2gb), `video card: vga ` just to dont get any trouble. `Virtual machine priority: higher than normal.` `Virtual disk: As much as you want for your cache`
   (2TB in my case) `Network: Default` `Iso file for bootup: the linux iso that you uploaded earlier. ` `Autostart: yes.`
5. Now start the VM and connect to it. And install linux

Configuring your linux mint installation
-

Open the network tab, wired and then click on the gear in the downright corner
Select IPv4  and change DHCP to Manual
Then Insert your choosen ip address for the lancache in my case it is "192.168.178.5" bc my
DCHP range goes from 192.168.178.20-200 and you dont want to interfere with your DCHP range.
As netmask go with default 24, honestly it is something about which ip ranges the network uses ect.
As Gateway go with your router IP so in my case it was 192.168.178.1
Then click apply
After that go back into the settings and disable IPv6 and click on apply
To activate your changes, just disable and enable your wired network.
Next, go to system settings -\> software sources -\> and change the mirror to your local mirror.
Then run
`~ $ sudo su`
`~ $ apt update && apt upgrade -y`
`~ $ apt install curl -y`
`~ $ curl -sSL https://get.docker.com/ | sh`
Check docker status with:
`~ $ systemctl status docker`
`~ $ apt install docker-compose`
`~ $ apt install git`

Installing lancache on the VM
-

`~ $ git clone https://github.com/lancachenet/docker-compose lancache`
`~ $ cd lancache`
`~/lancache $ nano .env`
`~/lancache $ sudo docker compose up -d`

**.env explained**

```
## See the "Settings" section in README.md for more details

## Set this to true if you're using a load balancer, or set it to false if you're using seperate IPs for each service.
## If you're using monolithic (the default), leave this set to true
USE_GENERIC_CACHE=true

## IP addresses that the lancache monolithic instance is reachable on
## Specify one or more IPs, space separated - these will be used when resolving DNS hostnames through lancachenet-dns. Multiple IPs can improve cache priming performance for some services (e.g. Steam)
## Note: This setting only affects DNS, monolithic and sniproxy will still bind to all IPs by default
LANCACHE_IP=<ip of lancache VM>

## IP address on the host that the DNS server should bind to
DNS_BIND_IP=<ip of lancache VM>

## DNS Resolution for forwarded DNS lookups
UPSTREAM_DNS=<router ip>

## Storage path for the cached data
## Note that by default, this will be a folder relative to the docker-compose.yml file
CACHE_ROOT=./lancache

## Change this to customise the size of the disk cache (default 2000g)
## If you have more storage, you'll likely want to increase this
## The cache server will prune content on a least-recently-used basis if it
## starts approaching this limit.
## Set this to a little bit less than your actual available space 
CACHE_DISK_SIZE=2000g

## Change this to allow sufficient index memory for the nginx cache manager (default 500m)
## We recommend 250m of index memory per 1TB of CACHE_DISK_SIZE 
CACHE_INDEX_SIZE=500m

## Change this to limit the maximum age of cached content (default 3650d)
CACHE_MAX_AGE=3650d

## Set the timezone for the docker containers, useful for correct timestamps on logs (default Europe/London)
## Formatted as tz database names. Example: Europe/Oslo or America/Los_Angeles
TZ=Europe/London
```

Installing Portainer agent on the VM and Pi
-

```
docker run -d \
  -p 9001:9001 \
  --name portainer_agent \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:2.19.4
```

Installing Portainer on your synology Nas
=

Please follow this exellent [guide](https://mariushosting.com/synology-30-second-portainer-install-using-task-scheduler-docker/) from Mariushosting

Setting up your Pi and installing Pihole, Docker, Docker compose, Portainer Agent
=

1. Download the latest image of Dietpi, and flash it with e.g. Balena etcher to your sd card.
2. Set your Pi ip to static
3. Follow the configuration process
4. SSH into your pi
5. `~ $ sudo apt update && sudo apt upgrade`
6. `~ $ sudo adduser [username]` & `~ $ sudo usermod -aG sudo [username]`
7. `~ $ sudo dietpi-software` browse software -\> select Docker and docker compose with spacebar
   exit. After that select install then back.

**The modified Pihole .yml file:**

```
version: '3.0'

services:
  pihole:
    container_name: pihole
    image: mwatz/pihole-dot-doh-updatelists-lancache-cache-domains:latest
    hostname: pihole
    domainname: pihole.local
    ports:
      - "443:443/tcp"
      - "53:53/tcp"
      - "53:53/udp"
      # - "67:67/udp"  # Uncomment this line if you need DHCP enabled
      - "80:80/tcp"
      - "853:853/tcp"
      - "853:853/udp"
    environment:
      - FTLCONF_LOCAL_IPV4=<IP address of device running the docker>
      - TZ=Europe/Berlin
      - WEBPASSWORD=<Password to access pihole>
      - WEBTHEME=default-dark
      - REV_SERVER=true
      - REV_SERVER_TARGET=<ip address of your router>
      - REV_SERVER_DOMAIN=localdomain
      - REV_SERVER_CIDR=<may be 192.168.1.0/24 if your router is 192.168.1.1>
      - PIHOLE_DNS_=127.1.1.1#5153;127.2.2.2#5253
      - DNSSEC=true
      - ADLISTS_URL=https://v.firebog.net/hosts/lists.php?type=tick
      - WHITELIST_URL=https://raw.githubusercontent.com/anudeepND/whitelist/master/domains/whitelist.txt
      - REGEX_BLACKLIST_URL=https://raw.githubusercontent.com/mmotti/pihole-regex/master/regex.list
    volumes:
      - './etc/pihole:/etc/pihole/:rw'
      - './etc/dnsmask:/etc/dnsmasq.d/:rw'
      - './etc/config:/config/:rw'
      - './etc/updatelists:/etc/pihole-updatelists/:rw'
      - './etc/lancache/config.json:/etc/cache-domains/config/config.json'
    cap_add:
      - NET_ADMIN
    restart: unless-stopped
```

But wait, dont run off and just create the .yml file. You **NEED** to create the directories first.
`~ $ mkdir pihole-setup && cd pihole-setup`.
Create a `docker-compose.yml`file inside this directory and paste and configure the provided .yml content into it. Now `~ $ mkdir -p etc/lancache etc/updatelists etc/config etc/dnsmask etc/pihole` then cd into the lancache folder like this `~ $ cd pihole-setup/etc/lancache` and create a config.json file `~ $ touch config.json` followed by `~ $ nano config.json` and paste this into it, dont forget to configure it to match your lancache ip.

example config.json
-

```
{
  "ips": {
    "generic":	["<insert here lancache ip>"]
  },
  "cache_domains": {
    "blizzard":     "generic",
    "epicgames":    "generic",
    "nintendo":     "generic",
    "origin":       "generic",
    "riot":         "generic",
    "sony":         "generic",
    "steam":        "generic",
    "uplay":        "generic",
    "wsus":         "generic"
  }
}
```

After all that hard work you can finally put that puppy into high gear with `~ $ docker compose up -d` if you now check portainer it should run. And your pihole admin interface should be reachable. You dont need to change any settings in pihole itself. **Important step:** you need to remove the lancache-dns service from you lancache host because you now use pihole as DNS service for lancache. Please dont forget to disable ipv6 on all pcs that you want to use lancache with. It currently only supports Ipv4.

Now please test your setup with an PC that uses the pihole DNS as its DNS. If youre on Windows dont forget to `ipconfig /flushdns ` if youre on linux you probably already know how to do it on your own. Next command is `nslookup lancache.steamcontent.com` . You Should get as output the IP of your Pihole and then Lancache.

Credits
=

- [Chat GPT](https://chat.openai.com) for just helping me going from no idea to a bit more idea
- [Lancache](https://lancache.net/) & Their awesome [Discord](https://discord.gg/BKnBS4u) community
- [Mariushosting](https://mariushosting.com/) for luring me into this mess with easy enough guides that you think you can handle it until you cant.
- [Mwatz container](https://hub.docker.com/r/mwatz/pihole-dot-doh-updatelists-lancache-cache-domains)
- [uklans example config](https://github.com/uklans/cache-domains/tree/master/scripts)
- If you like my work, [a donation to my world domination fund](https://paypal.me/donovandianchos?country.x=DE&locale.x=de_DE) is very much appreciated

Remember to disable your Lancache DNS,
