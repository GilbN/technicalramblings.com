---
title: "How to setup GitLab to use Rack Attack and ban abusive IPs and rate limit requests"
date: "2019-04-19"
categories: 
  - "backend"
  - "frontend"
  - "security"
tags: 
  - "fail2ban"
  - "gitlab"
  - "rack-attack"
  - "rate-limit"
  - "security"
coverImage: "gitlab-cover.png"
---

# {{ title }}

<img src="images/{{ coverImage}}"></img>

So, for anyone that's running the GitLab Docker container, I got real IPs to pass through to the logs and was then able to setup Rack Attack (similar to Fail2Ban), which is included by default, to ban abusive IPs and rate limit failed login attempts.

_**Note:** Rack attack does not ban IPs that fail to login on you GitLab, it bans IPs that fail to authenticate after x attempts when using git http authentication!_

The first step is getting real IPs to pass through to the GitLab container logs. The first thing that you will need to do is edit the **`gitlab.rb`** file from inside the container itself. You can do this by first execing into the container: **`docker exec -it gitlab /bin/bash`** Where `**gitlab**` is the container name, and then open the file with a text editor (vim is installed): `**:/# vim /etc/gitlab/gitlab.rb**` Next, you will need to find the following lines:

```
#nginx['real_ip_trusted_addresses'] = []
#nginx['real_ip_header'] = 
```

Uncomment the lines and then change them to something like this:

```
nginx['real_ip_trusted_addresses'] = ['172.17.0.0/16','172.18.0.0/16','172.19.0.0/16','172.20.0.0/16','192.168.1.0/24']
nginx['real_ip_header'] = 'X-Forwarded-For'
```

You will need to modify the IP address ranges to make your network(s). Once you're done, save the changes and exit the text editor. Now you need to reconfigure gitlab, while still inside the container, with the following command: **`gitlab-ctl reconfigure`** It should go through and reconfigure everything and add the changes you just made. You can verify that the changes were added with the following **`grep`** command: `**grep real_ip** **/var/opt/gitlab/nginx/conf/gitlab-http.conf**` The output should look similar to this:

```
:/# grep real_ip /var/opt/gitlab/nginx/conf/gitlab-http.conf
real_ip_header X-Forwarded-For;
set_real_ip_from 172.17.0.0/16;
set_real_ip_from 172.18.0.0/16;
set_real_ip_from 172.19.0.0/16;
set_real_ip_from 172.20.0.0/16;
set_real_ip_from 192.168.1.0/24;
```

With these changes now in place you should be able to see real IP addresses in the `**/log/nginx/gitlab_access.log**` the **`/log/gitlab-rails/application.log`** and the **`/log/gitlab-rails/production.log`** . I have `**/mnt/user/Docker/gitlab-ce**` set as my application data directory for the GitLab container so the full path for me is `**/mnt/user/Docker/gitlab-ce/log/nginx/gitlab_access.log**`.

Now that real IPs are getting logged you can enable and configure the built-in Rack Attack utility to find and block abusive IPs and rate limit failed login attempts. You will need to exec back into the container if you logged out: **`docker exec -it gitlab /bin/bash`** And open the **`gitlab.rb`** file again: **`:/# vim /etc/gitlab/gitlab.rb`** Find the following section and uncomment it:

```
# gitlab_rails['rack_attack_git_basic_auth'] = {
#   'enabled' => true,
#   'ip_whitelist' => ["127.0.0.1"],
#   'maxretry' => 5,
#   'findtime' => 60,
#   'bantime' => 3600
# }
```

You will need to add the IP ranges you wish to white-list, most likely the same list that you setup for the real\_ip configuration, so you don't accidentally ban yourself. It will end up looking something like this:

```
gitlab_rails['rack_attack_git_basic_auth'] = {
  'enabled' => true,
  'ip_whitelist' => ["127.0.0.1", "172.17.0.0/16", "172.18.0.0/16", "172.19.0.0/16", "172.20.0.0/16", "192.168.1.0/24"],
  'maxretry' => 5,
  'findtime' => 60,
  'bantime' => 3600
}
```

Next you need to reconfigure GitLab again: `**gitlab-ctl reconfigure**` Once the reconfiguration is done you can have someone or yourself test the functionality and try to git clone with invalid credentials. Once they're blocked, they should see a "**fatal: unable to access**" message like below.

```
gilbn@DESKTOP-UP9572O MINGW64 /f/github
$ git clone https://gitlab.domain.com/gilbn/testproject
Cloning into 'testproject'...
remote: HTTP Basic: Access denied
fatal: Authentication failed for 'https://gitlab.domain.com/gilbn/testproject.git/'

gilbn@DESKTOP-UP9572O MINGW64 /f/github
$ git clone https://gitlab.domain.com/gilbn/testproject
Cloning into 'testproject'...
remote: HTTP Basic: Access denied
fatal: Authentication failed for 'https://gitlab.domain.com/gilbn/testproject.git/'

gilbn@DESKTOP-UP9572O MINGW64 /f/github
$ git clone https://gitlab.domain.com/gilbn/testproject
Cloning into 'testproject'...
remote: Forbidden
fatal: unable to access 'https://gitlab.domain.com/gilbn/testproject/': The requested URL returned error: 403
```

In addition to them or you seeing that message , you can check the **`production.log`** file in the appdata directory for the container: **`grep -i blacklist /mnt/user/Docker/gitlab-ce/log/gitlab-rails/production.log`** If the IP was blocked, you should see something similar to the following:

```
root@Nostromo:# grep -i blacklist /mnt/user/Docker/gitlab-ce/log/gitlab-rails/production.log 
Rack_Attack: blacklist 46.246.123.33 GET /gilbn/testproject/info/refs?service=git-upload-pack
```

For rate limit throttling on failed logins you will see:

```
:~# grep -i rack_attack /home/gitlab/logs/gitlab-rails/production.log
Rack_Attack: throttle 82.37.176.202 POST /users/sign_in
```

And the user will see a **`Retry Later`** message in their browser.

Note: The rate limit does **not** follow the blacklist `**bantim**e`. The rate limit can be adjusted on the rate limit lines in the **`gitlab.rb`** file.

```
#gitlab_rails['rate_limit_requests_per_period'] = 10
#gitlab_rails['rate_limit_period'] = 60
```

Default is 10 requests in a 60 second period.

If you want to add fail2ban on your login page you can use this filter: **[https://gist.github.com/pawilon/238c278d3c6c4669771eb81b03264acd](https://gist.github.com/pawilon/238c278d3c6c4669771eb81b03264acd)**

Written by **Tronyx** - **[https://github.com/christronyxyocum](https://github.com/christronyxyocum)** Edited by **GilbN**

### If you need any extra help join the Discord server!

#### [![](https://img.shields.io/discord/591352397830553601.svg?style=for-the-badge&logo=discord "technicalramblings/theme.park!")](https://discord.gg/HM5uUKU)
