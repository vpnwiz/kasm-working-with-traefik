# Making Kasm Workspaces and Traefik work together

I read about [Kasm Workspaces](https://www.kasmweb.com) on [r/selfhosted](https://www.reddit.com/r/selfhosted/) and really wanted to try it out on my Ubuntu VPS. Problem is, I use Traefik for my reverse proxy and it's not technically supported by Kasm. There are also very few hits on searches for the two platforms together, and the information that's available is not very helpful. Maybe I'm the only person really interested in this 😂 but after a lot of time spent figuring it out I decided to write my own guide in case someone else is too. This assumes you already have Traefik setup and running on port 443. The files are available at [my Github project](https://github.com/vpnwiz/kasm-working-with-traefik). 

***Extra credit:*** My VPS is limited on resources, so I also figured out how to add all the CPU's and RAM from an old Mac Pro at home to Kasm as a remote agent, using Tailscale as the backend. It's another unsupported (very, lol) configuration but if there's interest I can write another guide on how I did that.

Disclaimer: I'm new to markdown, Github, Kasm and Nginx configs :) please correct if you see errors or a better way of doing something.

## The Traefik config
* Trying to do this with "normal" Traefik labels in the Kasm compose file ends up with a 400 Bad Request. The [loadbalancer.server.url](https://doc.traefik.io/traefik/routing/services/#servers-load-balancer) function is needed to make this work, and uses a file based config. 
* You can still use Traefik container labels on all your other services
* I followed [this guide](https://www.benjaminrancourt.ca/a-complete-traefik-configuration/) along with the Traefik reference docs to convert my docker-compose file into working [static](https://doc.traefik.io/traefik/reference/static-configuration/file/) and [dynamic](https://doc.traefik.io/traefik/reference/dynamic-configuration/file/) configuration files. This was actually a useful change as the traefik compose file was getting really big, glad I did it.
* My Traefik `docker-compose.yaml`, static `traefik.yaml`, and dynamic `config.yaml` files are at [the Github project](https://github.com/vpnwiz/kasm-working-with-traefik) for reference.
* If you're ready to just get to it, in your dynamic config file add the Kasm info: 
```
routers:
  rtr_kasm:
    entrypoints:
      - websecure # (whatever your Traefik 443 entrypoint from the static config file is named)
    service: svc_kasm
    rule: "Host(`kasm.yourdomain.com`)"
    tls:
      certresolver: letsencrypt # (or your certresolver reference, mine is le)

services:
  svc_kasm:
    loadBalancer:
      servers:
        - url: https://kasm_proxy:443
```


## Install Kasm Workspaces:
Kasm has really good documentation (except for Traefik integration lol). 
* We'll do a [single server installation](https://kasmweb.com/docs/latest/install.html) and also incorporate some of their [reverse proxy settings](https://kasmweb.com/docs/latest/how_to/reverse_proxy.html) to run Kasm on port 8443. Read up on it but don't run anything yet!

### Download and Install:
* Enter the first 3 commands to download and unpack Kasm:
	* `cd /tmp`
	* `curl -O https://kasm-static-content.s3.amazonaws.com/kasm_release_1.13.0.002947.tar.gz`
	* `tar -xf kasm_release_1.13.0.002947.tar.gz`
	* STOP HERE.
* Enter the following command. This installs Kasm into the default `/opt/kasm` dir, tells it to run on port 8443, skips the port and connection tests which will fail, and won't make it start up with a non-working config.
	* `sudo bash kasm_release/install.sh -e -L 8443 -D -B -N`
* ***MAKE SURE*** you save all the Kasm credentials and tokens shown at the end!

### Edit Kasm docker-compose:
* Download the docker-compose example file [on Github](https://github.com/vpnwiz/kasm-working-with-traefik) if you want a reference.
* Open `/opt/kasm/1.13.0/docker/docker-compose.yaml`
	* In the last container named "kasm_proxy", add: 
	```
	  expose:
	    - "443"
	  labels:
	    - traefik.enable=true
	  networks:
	    - kasm_default_network # already there
	    - <your traefik network> # so Traefik and Kasm can talk
	```
	At the bottom of the file under the last network section, add your Traefik network:
	```
	networks:
	  kasm_default_network:
	    external: true
	  <your traefik network>:
	    external: true
	```

### Replace Kasm's Nginx proxy config file
* Download ngingx.conf from [Github](https://github.com/vpnwiz/kasm-working-with-traefik)
* I keep my docker data and compose files in `/docker`, and to make this simple I added a volume to override the nginx.conf *inside* the container and instead, pull the new one from my docker dir: 
```
volumes:
  - /docker/kasm/nginx.conf:/etc/nginx/nginx.conf
```

## Now it's finally time to start Kasm services: 
`sudo /opt/kasm/current/bin/start`

## And the final change before it's ready:
* Login to your new Kasm install at https://kasm.yourdomain.com using admin@kasm.local and the password shown at the end of the install script. 
* [Follow the directions here](https://kasmweb.com/docs/latest/how_to/reverse_proxy.html#update-zones) to change the default zone.

# Start adding workspaces and enjoy using Kasm!


