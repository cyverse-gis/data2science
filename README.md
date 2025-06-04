# Data to Science
Data to Science (D2S) is a open-source web platform for visualizing and sharing of drone imagery which was developed by Ben Hancock and Jinha Jung at Purdue University. A production instance is hosted at Purdue University [https://ps2.d2s.org/](https://ps2.d2s.org/). 

## D2S Instance on Cyverse

Cyverse Tombstone VM https://tombstone-cloud.cyverse.org/; medium2 VM with 4 VCPUs, 16 gb ram, 

ssh ubuntu@128.196.65.95





## Software Architecture

The website code repository is [here](https://github.com/gdslab/data-to-science) D2S platform is a completely containerized web app which makes it easy to deploy with a relatively easy setup. The web app consists of 13 containers that are orchestrated using docker compose. These are known as 'services' within the `docker-compose.yml` file

* redis
* titiler
* **db** - postgis which is a spatial extension of postgresql database
* flower
* **backend** - ubuntu, python, conda, untwine(software to convert point clouds to _copc.laz_
* **pgadmin** - pgAdmin 4 is a web based administration tool for the PostgreSQL database
* **frontend** - bullseye_slim (minimalist debian linux); react
* celery_beat
* celery_worker
* proxy
* tusd
* pg_tileserv
* varnish
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
