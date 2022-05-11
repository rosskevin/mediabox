# :tv: Mediabox

![](https://github.com/rosskevin/mediabox/workflows/Multimedia%20Stack%20Deployment/badge.svg)

![](https://i.imgur.com/UZklu5w.jpg)

## TL;DR
A home media server using `docker-compose` that enables SSL wildcard certificates via Cloudflare and LetsEncrypt.  The approach allows layering of complexity and features including Single sign on (SSO) via GCP.

## What's in the stack?

NOTE: These are listed in the order I recommend they are layered and tested during setup.

### Baseline
<sub>`traefik-cloudflare.yml` and optional `traefik-oauth.yml`</sub>
* [Traefik](https://containo.us/traefik/) - Service ingress, routing, etc, handles SSL certificates via DNS-01 challenge and LetsEncrypt

### Misc
<sub>`misc.yml` and optional `misc-oauth.yml`</sub>
* [Watchtower](https://github.com/containrrr/watchtower) - A process for automating Docker container base image updates
* [Duplicati](https://www.duplicati.com/) - Backup software to store encrypted backups
* [Portainer](https://www.portainer.io/)

### Multimedia
<sub>`multimedia.yml`, optional `multimedia-oauth.yml` and `plex-hw.yml`</sub>
* [Prowlarr](https://wiki.servarr.com/prowlarr) - [quick start](https://wiki.servarr.com/prowlarr/quick-start-guide) - Indexer/manager manager/proxy that supports management of both Torrent Trackers and Usenet Indexers. It integrates seamlessly with Lidarr, Radarr, Readarr, and Sonarr offering complete management of your indexers with no per app Indexer setup required.  HINT: use local network names in settings e.g. http://prowlarr:9696, http://radarr:7878, http://sonarr:8989, http://sabnzbd:8080. 
* [SABnzbd](https://sabnzbd.org/) - Usenet downloader.  Use top level `DATA` env variable to avoid needing to specify "Remote Path" it the *arr applications.  This makes it easier to hardlink.  See #Layered setup stage 2
* [Sonarr](https://sonarr.tv/) - should pull indexer from prowlarr
* [Radarr](https://radarr.video/) - should pull indexer from prowlarr
* [Plex](https://www.plex.tv/) - NOTE: in initial config, connect to `http://<my-local-ip>:32400/web` first to complete the setup wizard.  Plex differs from every other container and uses `host` networking, therefore enable `Remote` access, port forward `32400` to `<my-local-ip>` and then you may access remotely via https://app.plex.tv.  Using `host` networking makes configuration easier, and ensures traffic runs unrestricted and local devices can easily discover the server.  The not-so-big-deal-downside is that plex.example.com no longer is your go-to url.
* [Organizr](https://github.com/causefx/Organizr) - Organize a single page with tabs to manage all services

### Optional additions
NOTE: updates likely needed here.
* [OpenVPN](https://github.com/dperson/openvpn-client) - vpn client
* [Deluge](https://deluge-torrent.org/) - torrent downloads

## Prerequisites
* [Docker](https://www.docker.com/)
* [Docker-Compose](https://docs.docker.com/compose/)

## Initial setup
1. Fork this repo (so you can keep changes if you need to)
1. Clone **_your_** forked repo
1. `cd mediabox`
1. `cp .env.template .env`
1. Replace variables on `.env` with whatever makes sense to you (follow the comments above each property).  Don't worry, start with the basics and get to the rest later

NOTE: set up duplicati.example.com later to make sure your repo and specifically your `.env` is backed up daily.

## Layered setup

Setting up a home server can be complex.  For all examples, we will assume you are using the domain `example.com`.  You usually need to setup your domain on cloudflare, forward ports on your router and set up dynamic dns.  Before attempting to configure additional applications, this repo is setup to allow you to layer in complexity by minimizing containers started using the `COMPOSE_FILE` variable.  The simple shell script wraps the `docker-compose` command to make it simple to execute `up`, `down`, `restart`, `logs`, `shell` and most any other typical `docker-compose` command (via `cmd`).  See [Usage](#usage) or simply execute `./mb` to see it.

## Layered setup stage 1

Successful completion of this stage means you can access both https://traefik.example.com and https://whoami.example.com, with optional IP whitelist restriction and/or optional Single Sign On.

1. **Set up cloudflare DNS** for your example.com domain
1. **Set up dynamic DNS** to cloudflare to update the `A` record for your example.com
1. **Add wildcard CNAME** - Add a `* CNAME` to `@` 
1. **Increase cloudflare security** - Configure increased security on your cloudflare zone, see [Cloudflare Settings for Traefik Docker: DDNS, CNAMEs, & Tweaks](https://www.smarthomebeginner.com/cloudflare-settings-for-traefik-docker/)
1. **Verify HTTPS connectivity** - In `.env` set `COMPOSE_FILE=traefik-cloudflare.yml`, `DOMAIN,SSL_ACME_EMAIL,CF_API_EMAIL,CF_API_KEY` and `./mb up` - verify connectivity to both https://traefik.example.com and https://whoami.example.com.  Check logs with `./mb logs traefik` and make sure there are no errors.  `./mb down` when done.
1. (optional) **Restrict IPs allowed** - Restrict your `IP_WHITELIST_SOURCERANGE` in `.env` to a minimal set of IP addresses, or if you want it public, leave it open by default `0.0.0.0/0`
1. (optional) **Setup Single Sign On** - Set `COMPOSE_FILE=traefik-cloudflare.yml:traefik-oauth.yml` in `.env` and `./mb up` - set up GCP based SSO via OAUTH. [Some background is available here](https://www.smarthomebeginner.com/google-oauth-with-traefik-2-docker/) on the external steps, but as-is you only need to make sure your associated ENV variables are populated.  Now verify the auth challenge to both https://traefik.example.com and https://whoami.example.com.  `./mb down` when done.

**All good?  If not do not continue.**

## Layered setup stage 2

Plan your disk layout and set up your NFS (or other) disk mounts (beyond the scope of this readme).  This setup follows best practices mentioned on [this article](https://wiki.servarr.com/Docker_Guide#The_Best_Docker_Setup) and [this reddit post](https://www.reddit.com/r/usenet/wiki/docker#wiki_the_best_docker_setup) to be able to use hardlinks and/or perform atomic `move` operations instead of `copy+delete` (which takes longer and requires more space).

TL;DR from [Trash guides "how to set up for docker"](https://trash-guides.info/How-to-setup-for/Docker/):

> The paths you use on the inside matter. Because of how Docker’s volumes work, passing in two or three volumes such as the commonly suggested /tv, /movies and /downloads makes them look like two or three file systems, even if they aren’t. This means hard links won’t work and instead of an instant move, a slower and more I/O intensive copy + delete is used.

```sh
# My disks layout:
#
# ├── nas
# │   └── mediabox (mounted as /data)
# |       ├── backups
# |       ├── torrents
# │       |   ├── movies
# │       |   ├── pictures
# │       |   └── tv
# |       └── usenet
# │       |   ├── movies
# │       |   ├── pictures
# │       |   └── tv
# |       └── media
# │           ├── movies
# │           ├── pictures
# │           └── tv
# └── ssd
#     ├── containers
#     └── repo
#
# CONTAINERS : Application docker container storage - SSD recommended.  Intend to backup this to the BACKUPS.
# DATA       : This directory will be the top level mount point mounted in containers.  Configure relative paths inside the applications.
# BACKUPS    : Target location for backups (can also use online services instead inside duplicati)
#
CONTAINERS=/containers
DATA=/mnt/nfs/mediabox
BACKUPS=/mnt/nfs/mediabox/backups
```

## Layered setup stage 3
Now it is time to determine what services you want to run.  

1. **Add primary applications** - Edit your `.env` and chain `mediabox.yml` on to your `COMPOSE_FILE` variable.  The same as above, `./mb up` then check the logs of the various containers like `./mb logs plex`.  You should now be able to access each by name e.g. https://plex.example.com. 
1. (optional) **Protect applications with SSO** - Edit your `.env` and chain `mediabox-oauth.yml` on to your `COMPOSE_FILE` variable.  Test again, but this time use an incognito browser (or other browser that has not utilized SSO in steps above) and be sure you receive a challenge for https://plex.example.com
1. (optional) **Torrents, books, etc** - view the remainder of the compose files and see what services you want.  Using the same methodology, chain the compose file, and test your changes. (I am not using torrents or books, so these may need adjustments, feel free to PR changes)

## Usage
```sh
~/mediabox$ ./mb
usage: mb [help|-h|--help] <subcommand>

optional arguments:
  help | -h | --help
    print this message and exit

subcommands:
  up
     starts the configured files
  down
     stops
  restart
     restarts, a combination of stop and start
  logs
     shows logs
  cmd <command>
     executes an arbitrary docker-compose command. Use "cmd help" to list them


The logs subcommand can be appended by flags and specify the container(s). example: 

  mb logs -f --tail 500 plex
    shows logs only for plex service


Be sure to run commands either with sudo, or as a user who is part of the "docker" group
```

## Updating
Watchtower automatically updates all apps (if docker image update is available) at 4 AM every day.

## VPN (may need updating)
With OpenVPN you can use any VPN provider following these steps:

1. Download your VPN OpenVPN config files (e.g: [NordVPN TCP/UDP config files](https://nordvpn.com/ovpn/))
2. Download your VPN CA file (e.g: [NordVPN CA & TLS key files](https://downloads.nordcdn.com/configs/archives/certificates/servers.zip))
3. Run the following (using NordVPN Brazil#65 as example)
```bash
# Copy required files
cp ~/Downloads/br65.nordvpn.com.udp.ovpn ${OPENVPN}/vpn.conf
cp ~/Downloads/br65_nordvpn_com_ca.crt ${OPENVPN}/vpn-ca.crt

# Write credentials
cat <<EOT >> ${OPENVPN}/vpn.auth
you@mail.com
YourVPNP4ssw0rD
EOT
```

## Credit
Much of this repo was developed by the original other and contributors at https://github.com/cristianmiranda/mediabox.  I did not choose to PR my changes because the repo was not very active.  While a PR from this repo could be created and applied upstream, I did not want to take the time to justify choices I made including some structural changes.  If someone wants to do that work to push this upstream, I welcome getting my changes contributed to the original body of work.

Anyway, thanks to @christianmiranda and other contributors from the original repo.
