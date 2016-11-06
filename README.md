# gisongcs
The motivation behind this repository is to outline the steps required to develop a geospatial application using Google Cloud Services (GCS). This is part of my PhD project whereby I am showcasing technology trends (such as Cloud, Big Data, Linked Data and others) for faster (and easier) development of Spatial Data Infastructures (SDIs) in Africa.

We are going to use <a href="http://kubernetes.io/" target="_blank">Kubernetes</a>, <a href="https://www.docker.com/" target="_blank">Docker containers</a>, <a href="http://geoserver.org/" target="_blank" >GeoServer</a>, a Linux operating system flavour such as <a href="https://www.debian.org/" target="_blank" >Debian</a>, <a href="https://openlayers.org/" target="_blank" >OpenLayers</a> and of course a DBMS with a spatial extender such as <a href="http://www.postgis.net/" target="_blank">PostGIS</a>.

So where do we need to start? A bit of an introduction to <a href="https://en.wikipedia.org/wiki/Cloud_computing" target="_blank">Cloud Computing</a> is ideal, and of course reference to <a href="https://cloud.google.com" target="_blank">Google Cloud Services</a>.

Step 1: connect to Google Cloud Shell. This is a shell environment for managing resources hosted on Google Cloud Platform. You will need a bit of Linux Skills to use this tool. See <a href="https://cloud.google.com/shell/docs/quickstart" target="_blank"> this page</a> for the getting started details.

Step 2: Setup the GCS compute zone. For example, issue the command 

gcloud config set compute/zone us-central1-b 

to set the compte zone to us-central1-b zone

Step 3: Pull (and/or create ) docker images. For this exrcise we pull two existing images: geoserver and postgis, and create the third one: NGINX Web Server. We create because we want to customise the web content. We pull because we want to resue existing images. Reuse saves time and money.

$ docker pull mdillon/postgis

$ docker pull kartoza/geoserver

$ docker build -t gcr.io/myphdprj/phd-nginx:v1 .

$ gcloud docker push gcr.io/myphdprj/phd-nginx:v1

The latter contains all the HTML files, javascripts, openlayer references etc. This is created from the following NGINX Dockerfile, assuming that the files are in a directory named webdir

FROM nginx

EXPOSE 80

COPY webdir /usr/share/nginx/html

Step 4: create disks

$ gcloud compute disks create --size 200GB postgis-disk

$ gcloud compute disks create --size 200GB geoserver-disk

Step 5: create the service files and replication controllers.

$ kubectl create -f mpostgres.yaml

$ kubectl create -f postgres-service.yaml

$ kubectl run phd-nginx --image=gcr.io/myphdprj/phd-nginx:v1 --port=80

$ kubectl expose rc (OR pod) phd-nginx --target-port=80 --type="LoadBalancer”

Step 5: The Pods (and services) should be running by now, on the specified ports. Use the commands below to inspect.

Useful commands

Listing what is currently running

$ kubectl get po

Executing interactive commands in a container

$ kubectl exec -ti postgis -- /bin/sh

Executing one-time commands in a container

$ kubectl exec postgis -- cat /etc/hostname

Viewing logs on a container

$ kubectl logs -f postgis

Step 6: Load the data

Load relevant data (administrative boundaries as polygons, schools as points, perfomance) into the postgresql RDBMS. This step involves creating a number of views that become "layers". Of course, take into account projection systems, Geometry columns, etc.

Step 7: Configure Geoserver.  Create layers, create styles, link the layers to styles, etc.

