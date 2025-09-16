# Data to Science
Data to Science (D2S) is a open-source web platform for visualizing and sharing of drone imagery which was developed by Ben Hancock and Jinha Jung at Purdue University. The website code repository is [https://github.com/gdslab/data-to-science](https://github.com/gdslab/data-to-science). D2S platform is a completely containerized web app which makes it easy to deploy with a relatively easy setup. One production instance is hosted at Purdue University [https://ps2.d2s.org/](https://ps2.d2s.org/) and another at University of Arizona Cyverse [https://d2s.cyverse.org](https://d2s.cyverse.org).

<br/>

## D2S Instance on Cyverse

The Cyverse instance of D2S is hosted on the thorn physical server `thorn.cyverse.org`

ssh -p 1657 <cyverse_user_name>@thorn.cyverse.org

Logging into thorn requires your Cyverse username & password. It also requires you to be on the Bio5 VPN if accessing from off campus


web app files are located in `/opt/data-to-science`

user-data storage is located at `/storage/`


The web app consists of a series of docker containers controlled by docker compose

Deploy docker compose
`docker compose -f docker-compose.prod.yml up -d`


<br/>



## Deployment Instructions
Instructions for launching the website on a linux machine is in the readme of this repo https://github.com/gdslab/data-to-science. 

### Nginx and Traffic Encryption
The web app has a two-tier proxy setup. We have nginx installed natively on the server (/etc/nginx), and also within the container. External internet traffic comes into the server at port 80 (default https port) or 443 (encrypted port) which is handled by the native nginx. It passes the request onto the containerized nginx (localhost:8004) which then sends the requests to different parts of the website. 


The nginx config file for d2s.cyverse.org is located in `/etc/nginx/sites-available/d2s.cyverse.org` on the host machine. 

There is a symbolic link at `/etc/nginx/sites-enabled`

`sudo ln -s /etc/nginx/sites-available/d2s.cyverse.org /etc/nginx/sites-enabled/`

### Traffic Encryption

Secure Sockets Layer (SSL) Certificate is a file that encrypts data transfer between a browser and a server. This is what makes `http` into `https`. In the nginx config file (`/etc/nginx/sites-available/d2s.cyverse.org`), you need to specify the certitication file `/etc/pki/tls/certs/cyverse.pem` and the cyverse.key `/etc/pki/tls/private/cyverse.key`

The ssl certificates were issued through Go Daddy and done by Cyverse DevOps guy Andy Edmonds. 




<br/>


### Other Nginx commands

Is nginx active and running?

`sudo systemctl status nginx`

Restart Nginx

`sudo systemctl restart nginx`

<br/>

Is nginx listening on port 80 (standard http port)?

`sudo lsof -i :80`


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

D2S uses a Python backend built with FastAPI, a lightweight web framework that adheres to modern standards for building APIs. The API serves as a central interface between client applications and a PostgreSQL database with the PostGIS extension, which manages the core application data and references to user-contributed datasets stored on the local file system. For long-running tasks initiated by client applications, D2S uses Redis as a message broker to enqueue tasks for Celery workers. When a worker becomes available, it executes the task asynchronously in the background. Task progress is tracked in the database and relayed to the clients by polling for updates. An example task is converting an uploaded GeoTIFF into a Cloud Optimized GeoTIFF (COG). After the file is received by the upload service and moved to file storage, the FastAPI application dispatches an asynchronous task to perform the conversion. A Celery worker handles the task in a background process, updating the file’s database record, and transferring the resulting COG to its final storage location.

Uploaded raster and point cloud data are stored in cloud-native formats to support efficient HTTP range requests, allowing clients to stream only the necessary portions. Raster data in Cloud Optimized GeoTIFF (COG) format is rendered as map tiles by a TiTiler service. Point cloud data is stored in the Cloud Optimized Point Cloud (COPC) format and visualized in an interactive 3D space using Potree, a WebGL-based viewer. Vector data is stored in PostgreSQL with PostGIS and served as Mapbox Vector Tiles (MVT) via the pg_tileserv service. All tile requests, raster or vector, are routed through a Varnish caching layer, which improves performance by serving cached responses and reducing backend load.

Access to user data is restricted through project-level role-based access control managed within the platform. All client requests for user data pass through authorization checks, and access to map tiles is granted through signed URLs that are time-limited and restricted to specific resources.

The D2S frontend web application is written in TypeScript using the React library and managed with Vite. The user interface and interaction design were developed in collaboration with a dedicated UI/UX team, guided by insights from user interviews. Styling is primarily handled with Tailwind CSS, a utility-first framework that enables highly customizable and consistent design. The design efforts focused on creating an intuitive interface tailored for geospatial researchers, enabling side-by-side view comparisons and supporting analytic workflows through user-centered navigation and clarity of visual feedback. The application uses React Router for navigation and organizes API communication through a centralized Axios-based module. Shared state is managed using React Context via the useContext hook. The central map interface for visualizing and interacting with user data is built with MapLibre GL JS, a TypeScript-based library that provides WebGL-powered map rendering.



<img src="https://github.com/gdslab/data-to-science/blob/main/docs/assets/system_overview.png" width=600>



* **frontend** - bullseye_slim (minimalist debian linux); react
* **backend** - ubuntu, python, conda, untwine(software to convert point clouds to _copc.laz_)
* **db** - postgis which is a spatial extension of postgresql database
* **pgadmin** - pgAdmin 4 is a web based administration tool for the PostgreSQL database. It is exposed at port `localhost:5050`. Login using the credentials specified in the _pgadmin_  service in the docker-compose.yml. To connect to the d2s database, use the credentials in the _db.env_ file (host=db, user=d2s, password=password)

<br/>
<br/>

* **redis** - Redis is serving as a message broker for Celery task queue system. It's the communication hub between your different Celery services.  
* **celery_beat** - Scheduler that triggers periodic tasks
* **celery_worker** - processes background tasks asynchronously
* **flower** - a web-based monitoring tool for Celery that lets you see task status, worker health, etc. Access at `localhost:5555`

Example flow: User uploads a large drone image. Backend says "I need this processed" and sends a message to Redis. Redis queues the message: {"task": "process_satellite_image", "file": "image.tif"}. A Celery worker picks up the message and processes the image. Results go back through Redis to your backend

<br/>

* **tusd** - tus is a protocol based on HTTP for resumable file uploads. Resumable means that an upload can be interrupted at any moment and can be resumed without re-uploading the previous data again. An interruption may happen willingly, if the user wants to pause, or by accident in case of an network issue or server outage.
  * TUSD_STORAGE - a bind mounted temporary storage for files that are in the process uploading. When uploaded, the files will go into _/home/ubuntu/data-to-science/tusd_storage_ . When finished processing, the data will disappear from this directory and be placed in the permanent storage directory at */var/lib/docker/volumes/data-to-science-ben_user-data/_data*. You can change the location of this permanet storage. It is called USER_DATA_STORAGE and can be specified in the _.env_ file. 


<br/>

* **titiler** - dynamic tiler for raster data
* **pg_tileserv** - pulls vector data (points, lines, polygons) from postgis database and serves vector map tiles in Mapbox Vector Tile (MVT) format
* **varnish** - HTTP cache layer for better performance

Vector tiles: PostGIS → pg_tileserv → Varnish (cache)

Raster tiles: GeoTIFF files → titiler → Varnish (cache)

<br/>

* **proxy** - The web app has a two-tier proxy setup. We have nginx installed nativel on the server, and also within the container. External internet traffic comes into the server at port 80/443 which is handled by the native nginx. It passes the request onto the containerized nginx which then sends the requests to different parts of the website. 


<br/>
<br/>

