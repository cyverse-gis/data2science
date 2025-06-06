# Data to Science
Data to Science (D2S) is a open-source web platform for visualizing and sharing of drone imagery which was developed by Ben Hancock and Jinha Jung at Purdue University. The website code repository is [https://github.com/gdslab/data-to-science](https://github.com/gdslab/data-to-science). D2S platform is a completely containerized web app which makes it easy to deploy with a relatively easy setup. One production instance is hosted at Purdue University [https://ps2.d2s.org/](https://ps2.d2s.org/) and another at University of Arizona Cyverse [https://d2s.cyverse.org](https://d2s.cyverse.org).

<br/>

## D2S Instance on Cyverse

The Cyverse instance of D2S is hosted on a Tombstone VM https://tombstone-cloud.cyverse.org/; medium2 VM with 4 VCPUs, 16 gb ram.  

Login to server `ssh ubuntu@128.196.65.95`. You need to have public ssh keys on the host VM. 

If using VsCode, you can add this to the .ssh config file to one-click your way on to the VM

```
Host d2s
  HostName 128.196.65.95
  User ubuntu
  IdentityFile ~/.ssh/id_rsa
  ServerAliveInterval 120
  ForwardX11 yes
```

The website git repository is at `/home/ubuntu/data-to-science` which is synced with https://github.com/gdslab/data-to-science

<br/>



## Deployment Instructions
Instructions for launching the website on a linux machine is in the readme of this repo https://github.com/gdslab/data-to-science. 

### Nginx and Traffic Encryption
Once the containers have been built and launched (`docker compose up -d`), the host machine needs nginx installed natively to act as web server and reverse proxy. This is needed so internet traffic that goes to d2s.cyverse.org will get to the web app. A reverse proxy acts as gatekeeper or middleman to handle web requests. A request will come into to port 80 (default http port) or 443 (encrypted port). Nginx listens to these ports for requests and then sends the request on to localhost:8000 to meet the request.

Install `nginx`

```
sudo apt install nginx
```



The following nginx config file (d2s.cyverse.org) should be placed in `/etc/nginx/sites-available` on the host machine:

```
server {
        server_name d2s.cyverse.org;

        access_log /var/log/nginx/d2s.cyverse.org-access.log;
        error_log  /var/log/nginx/d2s.cyverse.org-error.log;

        client_max_body_size 0;
        client_body_timeout 300s;
        keepalive_timeout 600s;

        proxy_http_version 1.1;
        proxy_request_buffering off;

        location / {
                proxy_pass http://localhost:8000;

                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
        }
}
```

Then on the CLI, create a symbolic link at `/etc/nginx/sites-enabled`

`sudo ln -s /etc/nginx/sites-available/d2s.cyverse.org /etc/nginx/sites-enabled/`

### Other Nginx commands

Is nginx active and running?

`sudo systemctl status nginx`

Restart Nginx

`sudo systemctl restart nginx`

<br/>

Is nginx listening on port 80 (standard http port)?

`sudo lsof -i :80`

### Traffic Encryption

Secure Sockets Layer (SSL) Certificate is a file that encrypts data transfer between a browser and a server. This is what makes `http` into `https`. 

We are using [Let's Encrypt](https://letsencrypt.org/) to secure our address. 

```
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d d2s.cyverse.org
```

* Edits your Nginx config to include SSL settings

* Gets and installs the certificate

* Enables HTTPS

* The certificate expires after 90 days, but is automatically renewed via systemd or cron

  * The certificate and private key (in /etc/letsencrypt/live/d2s.cyverse.org/)

  * A renewal config file (/etc/letsencrypt/renewal/d2s.cyverse.org.conf)

<br/>




### Software Architecture
D2S platform is a completely containerized web app which makes it easy to deploy with a relatively easy setup. The web app consists of 13 containers that are orchestrated using docker compose. These are known as 'services' within the `docker-compose.yml` file

* **redis**
* **titiler**
* **db** - postgis which is a spatial extension of postgresql database
* **flower**
* **backend** - ubuntu, python, conda, untwine(software to convert point clouds to _copc.laz_)
* **pgadmin** - pgAdmin 4 is a web based administration tool for the PostgreSQL database
* **frontend** - bullseye_slim (minimalist debian linux); react
* **celery_beat**
* **celery_worker**
* **proxy** - nginx reverse proxy web server
* **tusd** - 
* **pg_tileserv**
* **varnish**
* 



<br/>
<br/>

## Data2Science Python 

D2S has a python library called [d2spy](https://py.d2s.org) that allows you to bring data stored in D2S directly into a python environment (e.g., jupyter notebook). This repo contains example juypter notebooks that use D2S as a starting point for a machine learning workflow.

<br>

### Run locally

clone this repository
`git clone https://github.com/jeffgillan/data_to_science.git`

`cd data_to_science`

Create a new conda environment and install the software necessary for the code to run (d2s.py)

`conda env create --file lettuce_detecto.yml`

`conda activate lettuce_detecto`
