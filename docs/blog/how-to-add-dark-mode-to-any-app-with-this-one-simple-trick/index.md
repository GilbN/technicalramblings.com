---
title: "How to add dark mode to any app with this one simple trick!"
date: "2021-01-03"
categories: 
  - "backend"
  - "frontend"
tags: 
  - "containers"
  - "dark-theme"
  - "docker"
  - "docker_mods"
  - "letsencrypt"
  - "linuxserver"
  - "nginx"
  - "proxy"
  - "reverse"
  - "skins"
  - "swag"
  - "theme-park"
  - "themes"
  - "unraid"
coverImage: "raw-32-2.jpg"
---

# {{ title }}

<img src="images/{{ coverImage}}"></img>

Developers hate me for sharing this one weird trick!

Yes..the title is clickbait, and **no** this does not work on any app/container ;)

I have for some time now been creating **[css themes/skins](https://github.com/gilbN/theme.park)** for different applications that reside in the "media server/selfhosting" category. Normally you would add the themes using a "subfilter" module like [http://nginx.org/en/docs/http/ngx\_http\_sub\_module.html](http://nginx.org/en/docs/http/ngx_http_sub_module.html) or [https://httpd.apache.org/docs/2.4/mod/mod\_filter.html](https://httpd.apache.org/docs/2.4/mod/mod_filter.html). This means that you would have to reverse proxy the application to be able to add the theme. Doing that would only apply the theme when accessing the application though the proxy and not locally.

So, to fix that I've created a bunch of [docker mods](https://blog.linuxserver.io/2019/09/14/customizing-our-containers/) for all the applications that have a **[linuxserver](https://www.linuxserver.io/)** container. The docker mod, will run a script at startup to inject the stylesheet into the html file using [sed](https://www.gnu.org/software/sed/manual/sed.html). The script does the exact same thing as the subfiltering module except it does it on the backend instead of on the proxy side. You can find all the mods here: [https://github.com/gilbN/theme.park/tree/docker-mods](https://github.com/gilbN/theme.park/tree/docker-mods)

The scripts are quite simple and only has a couple of variables. `TP_DOMAIN` and `TP_THEME` Both these variable have a default value if not set. `TP_DOMAIN` is set to `theme-park.dev` and `TP_THEME` is set to `organizr_dark`

Example:

```bash
#!/usr/bin/with-contenv bash

echo '---------------------------'
echo '|  Sonarr theme.park Mod  |'
echo '---------------------------'

# Display variables for troubleshooting
echo -e "Variables set:\\n\
'TP_DOMAIN'=${TP_DOMAIN}\\n\
'TP_THEME'=${TP_THEME}\\n"

# Set default
if [[ -z ${TP_DOMAIN} ]]; then
    echo 'No domain set, defaulting to theme-park.dev'
    TP_DOMAIN='theme-park.dev'
fi

if [[ -z ${TP_THEME} ]]; then
    echo 'No theme set, defaulting to organizr-dark'
    TP_THEME='organizr-dark'
fi

# Adding stylesheets
if ! grep -q "${TP_DOMAIN}" /app/sonarr/bin/UI/index.html; then
    echo '---------------------------'
    echo '|  Adding the stylesheet  |'
    echo '---------------------------'
    sed -i "s/<\/head>/<link rel='stylesheet' href='https:\/\/${TP_DOMAIN}\/theme.park\/CSS\/themes\/sonarr\/${TP_THEME}.css'><\/head> /g" /app/sonarr/bin/UI/index.html
    printf 'Stylesheet set to %s\n' "${TP_THEME}
    "
fi
```

## Adding the theme

Adding the mod, is quite simple. Add the variable `DOCKER_MODS=ghcr.io/gilbn/theme.park:<app>` e.g. `ghcr.io/gilbn/theme.park:sonarr` to your docker run command or compose file.

On Unraid simply click on `+ Add another Path, Port, Variable, label or Device`

[![](images/chrome_FjpV8pDRDU.png)](https://technicalramblings.com/wp-content/uploads/2021/01/chrome_FjpV8pDRDU.png) 

And fill out the fields.

[![](images/chrome_bcil72yAkv.png)](https://technicalramblings.com/wp-content/uploads/2021/01/chrome_bcil72yAkv.png)

As noted earlier it will default to the **[organizr\_dark](https://github.com/gilbN/theme.park/wiki/Organizr-Dark)** theme and the `theme-park.dev` domain. So if you want a different theme, you must add the `TP_THEME` variable with the value you want. The available themes are: `aquamarine`, `hotline`, `space-gray`,`dark`, `plex` and `organizr`

Click the banners here for screenshots of the different themes: [https://docs.theme-park.dev/#themes](https://docs.theme-park.dev/#themes)

You can find applications that support Docker Mods installation here: [https://github.com/gilbN/theme.park/tree/docker-mods](https://github.com/gilbN/theme.park/tree/docker-mods) For other installation methods check the wiki [https://docs.theme-park.dev/setup/](https://docs.theme-park.dev/setup/)
