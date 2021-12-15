---
title: "Blocking SSH Connections with the GeoLite2 Database"
date: "2020-04-28"
categories: 
  - "security"
tags: 
  - "geoblock"
  - "geolite2"
  - "remote"
  - "security"
  - "ssh"
coverImage: "arget-zvHhKiVuR9M-unsplash-scaled.jpg"
authors:
    - Tronyx
---

# {{ title }}

<img src="images/{{ coverImage}}"></img>

This guide will make it possible to block SSH access to your server based on the origin of the IP trying to access it!

First and foremost, I need to give credit where it is due. This method is a modified and combined version of these two approaches to achieving this, so most of the credit goes to them:

- [https://www.claudiokuenzler.com/blog/676/ssh-access-filter-based-on-geoip-database-allow-deny](https://www.claudiokuenzler.com/blog/676/ssh-access-filter-based-on-geoip-database-allow-deny)
- [https://www.axllent.org/docs/view/ssh-geoip/](https://www.axllent.org/docs/view/ssh-geoip/)

## Setting up the MaxMind GeoLite2 Database

The first thing we need to do is get yourself a copy of the MaxMind GeoLite2 Database. This will allow the detection of the Country of origin of IP addresses that try to access your Server. You can follow the first bit of instructions from the post “Blocking countries with GeoLite2 in Nginx using the LetsEncrypt Docker container” found [here](https://technicalramblings.com/blog/blocking-countries-with-geolite2-using-the-letsencrypt-docker-container/).

As the database gets updated regularly, it’s a good idea to automate downloading the database. The above linked post covers that for Unraid, so you can follow those instructions. If you’re not using Unraid, just create the script and create a cronjob manually.

## Setting Up GeoIP Blocking for SSH

Now that the database for the IP lookup is in place, we can move on to configuring the use of the lookup to deny SSH access to the Server based on the Country the IP is associated with.

We will need to install two packages to get this all working. The first one is the command-line tool for performing the IP lookups, `mmdb-bin`. The second one is a modified grep utility, called `grepcidr`, for CIDR notated IP addresses, IE: `192.168.1.0/24`. On Ubuntu 18.04, you can do this with the following command:

`apt install mmdb-bin grepcidr`

Next, we will setup the script that performs the lookup of the IP address that is attempting to access your Server to determine the Country that it’s coming from. Create the script wherever you would like, but make sure it’s easy to remember as we will need it for another step, IE: `/home/tronyx/scripts/ssh_filter.sh`. Paste in the following code to create the script:

```bash
#!/usr/bin/env bash

# UPPERCASE space-separated country codes to ACCEPT
ALLOW_COUNTRIES="US" # Make sure you add your Country code here
# Space-separated list of IPs to whitelist
# You can use CIDR notation as well, IE: 192.168.1.0/24
ALLOW_IPS="" # You can add your public IP address here

if [[ $# -ne 1 ]]; then
  echo "Usage:  `basename $0` <ip>" 1>&2
  exit 0 # return true in case of config issue
fi

# Fixed static IP
echo $1 | /usr/bin/grepcidr "$ALLOW_IPS" &> /dev/null

if [ ${PIPESTATUS[1]} -eq 0 ]; then
  RESPONSE="ALLOW"
  logger "$RESPONSE sshd connection from $1 (Whitelisted)"
  exit 0
fi

COUNTRY="$(/usr/bin/mmdblookup -f /home/letsencrypt-vps/config/geolite2/GeoLite2-City.mmdb -i "$1" country iso_code 2>&1| awk -F '"' '{ print $2 }'|head -n 2|tail -n 1)"

[[ $COUNTRY = "IP Address not found" || $ALLOW_COUNTRIES =~ $COUNTRY ]] && RESPONSE="ALLOW" || RESPONSE="DENY"

if [ $RESPONSE = "ALLOW" ]
then
  exit 0
else
  logger "$RESPONSE sshd connection from $1 ($COUNTRY)"
  exit 1
fi
```

Make sure to put the country codes into the allow for any Country that you want to allow access from, especially the Country that you live in, so that you do not lose access to your Server. You can find a list of all of the country codes [HERE](https://dev.maxmind.com/geoip/legacy/codes/iso3166/). If you would like, you can add your public IP address to the whitelist, if your server is remote, to ensure you never get blocked, but you shouldn’t be blocking the Country where you live anyway. Make the script executable with the following command:

`chmod a+x /home/tronyx/scripts/ssh_filter.sh`

## Locking Down SSH

The final thing that we need to do is lock SSH down by telling it what IP addresses to deny and which ones to allow, determined by the script that we just created. Add the following to the `/etc/hosts.deny` file: `sshd: ALL` This will initially block all connections to the Server via SSH. Then, add the following to the `/etc/hosts.allow` file:

`sshd: ALL: aclexec /home/tronyx/scripts/ssh_filter.sh %a`

This part tells it to execute the `ssh_filter.sh` script against the IP address that is attempting to connect to the Server via SSH to determine whether or not the IP address is allowed to connect. If the lookup of the IP address fails to determine the Country it’s from, like a private IP address for example, it will allow the connection.

## Time to Test

Now that we’ve got all of the requirements in place, we can test the script to make sure that it’s doing what it’s supposed to. We can do this by running the script against some IP addresses from Countries that we’ve configured the script to block and allow. I am located in the US, so I’m going to first test the IP of one of Google’s DNS Servers:

`/home/tronyx/scripts/ssh_filter.sh 8.8.8.8`

The script will not output anything, and it’s only configured to log deny and whitelist messages to the `/var/log/syslog` file, so you want to make sure that you do not see a deny message for the IP address you are testing. Next, as I’m only allowing IP addresses from the US, I’m going to test an IP address from China and one from Germany to make sure I see a deny message in the syslog for both of them:

`/home/tronyx/scripts/ssh_filter.sh 222.186.180.130 /home/tronyx/scripts/ssh_filter.sh 95.217.185.78`

Looking at the at the `/var/log/syslog` file, I see that both IP addresses were denied:

```log
Apr 26 12:05:04 1and1-vps root: DENY sshd connection from 222.186.180.130 (CN)
Apr 26 12:07:11 1and1-vps tronyx: DENY sshd connection from 95.217.185.78 (DE)
```

A user that was trying to connect to your Server via SSH from a Country that you’re blocking would simply see that the connection was closed or that it was unable to be established, like this:

`ssh_exchange_identification: Connection closed by remote host`

When accessing SSH from an IP address that you’ve added to the whitelist, the entry in the `/var/log/syslog` will look like this:

```log
Apr 26 12:07:30 1and1-vps tronyx: ALLOW sshd connection from 192.168.1.2 (Whitelisted)
```

Congratulations, your Server is now denying SSH connections based on the Country in which the IP address is associated!
