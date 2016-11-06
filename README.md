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

The set of files (YAML files) and other config pieces are outlined below.

geoserver.yaml: This script creates the GeoServer pod

apiVersion: v1
kind: Pod
metadata:
  name: geoserver
  labels:
    name: geoserver
spec:
  containers:
    - image: kartoza/geoserver
      name: geoserver
      ports:
        - containerPort: 8080
          name: geoserver
      volumeMounts:
          # Name must match the volume name below.
        - name: geoserver-persistent-storage
          # Mount path within the container.
          mountPath: /opt/geoserver/data_dir
  volumes:
    - name: geoserver-persistent-storage
      gcePersistentDisk:
        # This GKE persistent disk must already exist.
        pdName: geoserver-disk
        fsType: ext4

geoserver-service.yaml: This script creates the Geoserver service

apiVersion: v1
kind: Service
metadata:
  labels:
    name: gsfrontend
  name: gsfrontend
spec:
  type: LoadBalancer
  ports:
    # The port that this service should serve on.
    - port: 8080
      targetPort: 8080
      protocol: TCP
  # Label keys and values that must match in order to receive traffic for this service.
  selector:
    name: geoserver

postgres.yaml: This script creates the PostGIS pod

apiVersion: v1
kind: Pod
metadata:
  name: postgis
  labels:
    name: postgis
spec:
  containers:
    - resources:
        limits:
          cpu: 0.5
      image: mdillon/postgis
      name: postgis
      env:
        - name: POSTGRES_USER
          value: postgres
        - name: POSTGRES_PASSWORD
          value: xxxxxxxxx
        - name: DB_PASS
          value: xxxxxxxxx
        - name: POSTGRES_DB
          value: gis
        - name: PGDATA
          value: /opt/postgis/data_dir
      ports:
        - containerPort: 5432
          name: postgis
      volumeMounts:
          # This name must match the volumes.name below.
        - name: postgis-persistent-storage
          mountPath: /opt/postgis
  volumes:
    - name: postgis-persistent-storage
      gcePersistentDisk:
        # This disk must already exist.
        pdName: postgis-disk
        fsType: ext4

postgres-service.yaml: This script creates the PostGIS service

apiVersion: v1
kind: Service
metadata:
  labels:
    name: psfrontend
  name: psfrontend
spec:
  ports:
    # The port that this service should serve on.
    - port: 5432
  # Label keys and values that must match in order to receive traffic for this service.
  selector:
    name: postgis

auto.sh: A Linux script to automate creation of containers, pods and services
!/bin/sh
gcloud config set compute/zone us-central1-b
gcloud container clusters create gscont --num-nodes 3
gcloud compute disks create --size 200GB postgis-disk
gcloud compute disks create --size 200GB geoserver-disk
kubectl create -f mpostgres.yaml
kubectl create -f postgres-service.yaml
kubectl create -f geoserver.yaml
kubectl create -f geoserver-service.yaml
kubectl get services gsfrontend psfrontend

cleanup.sh: A Linux script to automate clean-up of containers, pods and services
!/bin/sh
gcloud config set compute/zone us-central1-b
kubectl delete service gsfrontend
kubectl delete service psfrontend
kubectl delete pod geoserver
kubectl delete pod postgis
gcloud container clusters delete gscont
gcloud compute disks delete postgis-disk geoserver-disk
Accessing GeoServer in the cloud
First use the GKE to inspect the services to determine the IP Address:

$ kubectl get services gsfrontend psfrontend
NAME         CLUSTER-IP       EXTERNAL-IP    PORT(S)    AGE
gsfrontend   000.000.000.000   000.000.000.000   8080/TCP   5h
kubernetes   000.000.000.000     <none>         443/TCP    1d
psfrontend   000.000.000.000   <none>         5432/TCP   5h


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

Load relevant data (administrative boundaries as polygons, schools as points, perfomance) into the postgresql RDBMS. This step involves creating a number of views that become "layers".

Step 7: Configure GorServer. 

Create styles, for example

<StyledLayerDescriptor xmlns="http://www.opengis.net/sld" xmlns:ogc="http://www.opengis.net/ogc" xmlns:xlink="http://www.w3.org/1999/xlink" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" version="1.0.0" xsi:schemaLocation="http://www.opengis.net/sld StyledLayerDescriptor.xsd">
	<NamedLayer>
		<Name>count_oranges</Name>
		<UserStyle>
			<Title>count_oranges</Title>
			<FeatureTypeStyle>
				<Rule>
					<Name>lt_eq_3</Name>
					<Title>&lt;=3</Title>
					<ogc:Filter>
						<ogc:PropertyIsLessThanOrEqualTo>
							<ogc:PropertyName>count</ogc:PropertyName>
							<ogc:Literal>3</ogc:Literal>
						</ogc:PropertyIsLessThanOrEqualTo>
					</ogc:Filter>
					<PolygonSymbolizer>
						<Fill>
							<CssParameter name="fill">#feedde</CssParameter>
						</Fill>
						<Stroke>
							<CssParameter name="stroke">#FFFFFF</CssParameter>
							<CssParameter name="stroke-width">1</CssParameter>
						</Stroke>
					</PolygonSymbolizer>
                    <TextSymbolizer>
                        <Geometry>
                          <ogc:Function name="centroid">
                            <ogc:PropertyName>geom</ogc:PropertyName>
                          </ogc:Function>
                        </Geometry>
         				<Label>
           					<ogc:PropertyName>county_nam</ogc:PropertyName>
         				</Label>
         				<Fill>
           					<CssParameter name="fill">#000000</CssParameter>
         				</Fill>
                        <Font>
           					<CssParameter name="font-family">Arial</CssParameter>
           					<CssParameter name="font-size">9</CssParameter>
           					<CssParameter name="font-style">normal</CssParameter>
         				</Font>
                        <LabelPlacement>
                           <PointPlacement>
                             <AnchorPoint>
                               <AnchorPointX>0.5</AnchorPointX>
                               <AnchorPointY>0.0</AnchorPointY>
                             </AnchorPoint>
                           </PointPlacement>
                         </LabelPlacement>
       				</TextSymbolizer>
				</Rule>
				<Rule>
					<Name>gte_3_lt_6</Name>
					<Title>&gt;3 and &lt;=6</Title>
					<ogc:Filter>
						<ogc:And>
							<ogc:PropertyIsGreaterThan>
								<ogc:PropertyName>count</ogc:PropertyName>
								<ogc:Literal>3</ogc:Literal>
							</ogc:PropertyIsGreaterThan>
							<ogc:PropertyIsLessThanOrEqualTo>
								<ogc:PropertyName>count</ogc:PropertyName>
								<ogc:Literal>6</ogc:Literal>
							</ogc:PropertyIsLessThanOrEqualTo>
						</ogc:And>
					</ogc:Filter>
					<PolygonSymbolizer>
						<Fill>
							<CssParameter name="fill">#fdbe85</CssParameter>
						</Fill>
							<Stroke>
								<CssParameter name="stroke">#FFFFFF</CssParameter>
							<CssParameter name="stroke-width">1</CssParameter>
						</Stroke>
					</PolygonSymbolizer>
                    <TextSymbolizer>
                                              <Geometry>
                          <ogc:Function name="centroid">
                            <ogc:PropertyName>geom</ogc:PropertyName>
                          </ogc:Function>
                        </Geometry>
         				<Label>
           					<ogc:PropertyName>county_nam</ogc:PropertyName>
         				</Label>
         				<Fill>
           					<CssParameter name="fill">#000000</CssParameter>
         				</Fill>
                                              <Font>
           					<CssParameter name="font-family">Arial</CssParameter>
           					<CssParameter name="font-size">9</CssParameter>
           					<CssParameter name="font-style">normal</CssParameter>
         				</Font>
                                              <LabelPlacement>
                           <PointPlacement>
                             <AnchorPoint>
                               <AnchorPointX>0.5</AnchorPointX>
                               <AnchorPointY>0.0</AnchorPointY>
                             </AnchorPoint>
                           </PointPlacement>
                         </LabelPlacement>
       				</TextSymbolizer>
				</Rule>
				<Rule>
					<Name>gte_6_lt_9</Name>
					<Title>&gt;6 and &lt;=9</Title>
					<ogc:Filter>
						<ogc:And>
							<ogc:PropertyIsGreaterThan>
								<ogc:PropertyName>count</ogc:PropertyName>
								<ogc:Literal>6</ogc:Literal>
							</ogc:PropertyIsGreaterThan>
							<ogc:PropertyIsLessThanOrEqualTo>
								<ogc:PropertyName>count</ogc:PropertyName>
								<ogc:Literal>9</ogc:Literal>
							</ogc:PropertyIsLessThanOrEqualTo>
						</ogc:And>
					</ogc:Filter>
					<PolygonSymbolizer>
						<Fill>
							<CssParameter name="fill">#fd8d3c</CssParameter>
						</Fill>
						<Stroke>
							<CssParameter name="stroke">#FFFFFF</CssParameter>
							<CssParameter name="stroke-width">1</CssParameter>
						</Stroke>
					</PolygonSymbolizer>
                    <TextSymbolizer>
                                              <Geometry>
                          <ogc:Function name="centroid">
                            <ogc:PropertyName>geom</ogc:PropertyName>
                          </ogc:Function>
                        </Geometry>
         				<Label>
           					<ogc:PropertyName>county_nam</ogc:PropertyName>
         				</Label>
         				<Fill>
           					<CssParameter name="fill">#000000</CssParameter>
         				</Fill>
                                              <Font>
           					<CssParameter name="font-family">Arial</CssParameter>
           					<CssParameter name="font-size">9</CssParameter>
           					<CssParameter name="font-style">normal</CssParameter>
         				</Font>
                                              <LabelPlacement>
                           <PointPlacement>
                             <AnchorPoint>
                               <AnchorPointX>0.5</AnchorPointX>
                               <AnchorPointY>0.0</AnchorPointY>
                             </AnchorPoint>
                           </PointPlacement>
                         </LabelPlacement>
       				</TextSymbolizer>
				</Rule>
				<Rule>
					<Name>gte_9_lt_12</Name>
					<Title>&gt;9 and &lt;=12</Title>
					<ogc:Filter>
						<ogc:And>
							<ogc:PropertyIsGreaterThan>
								<ogc:PropertyName>count</ogc:PropertyName>
								<ogc:Literal>9</ogc:Literal>
							</ogc:PropertyIsGreaterThan>
							<ogc:PropertyIsLessThanOrEqualTo>
								<ogc:PropertyName>count</ogc:PropertyName>
								<ogc:Literal>12</ogc:Literal>
							</ogc:PropertyIsLessThanOrEqualTo>
						</ogc:And>
					</ogc:Filter>
					<PolygonSymbolizer>
						<Fill>
							<CssParameter name="fill">#e6550d</CssParameter>
						</Fill>
						<Stroke>
							<CssParameter name="stroke">#FFFFFF</CssParameter>
							<CssParameter name="stroke-width">1</CssParameter>
						</Stroke>
					</PolygonSymbolizer>
                    <TextSymbolizer>
                                              <Geometry>
                          <ogc:Function name="centroid">
                            <ogc:PropertyName>geom</ogc:PropertyName>
                          </ogc:Function>
                        </Geometry>
         				<Label>
           					<ogc:PropertyName>county_nam</ogc:PropertyName>
         				</Label>
         				<Fill>
           					<CssParameter name="fill">#000000</CssParameter>
         				</Fill>
                                              <Font>
           					<CssParameter name="font-family">Arial</CssParameter>
           					<CssParameter name="font-size">9</CssParameter>
           					<CssParameter name="font-style">normal</CssParameter>
         				</Font>
                                              <LabelPlacement>
                           <PointPlacement>
                             <AnchorPoint>
                               <AnchorPointX>0.5</AnchorPointX>
                               <AnchorPointY>0.0</AnchorPointY>
                             </AnchorPoint>
                           </PointPlacement>
                         </LabelPlacement>
       				</TextSymbolizer>
				</Rule>
				<Rule>
					<Name>gt_12</Name>
					<Title>&gt;12</Title>
					<ogc:Filter>
						<ogc:PropertyIsGreaterThan>
							<ogc:PropertyName>count</ogc:PropertyName>
							<ogc:Literal>12</ogc:Literal>
						</ogc:PropertyIsGreaterThan>
					</ogc:Filter>
					<PolygonSymbolizer>
						<Fill>
							<CssParameter name="fill">#a63603</CssParameter>
						</Fill>
						<Stroke>
							<CssParameter name="stroke">#FFFFFF</CssParameter>
							<CssParameter name="stroke-width">1</CssParameter>
						</Stroke>
					</PolygonSymbolizer>
                    <TextSymbolizer>
                                              <Geometry>
                          <ogc:Function name="centroid">
                            <ogc:PropertyName>geom</ogc:PropertyName>
                          </ogc:Function>
                        </Geometry>
         				<Label>
           					<ogc:PropertyName>county_nam</ogc:PropertyName>
         				</Label>
         				<Fill>
           					<CssParameter name="fill">#000000</CssParameter>
         				</Fill>
                                              <Font>
           					<CssParameter name="font-family">Arial</CssParameter>
           					<CssParameter name="font-size">9</CssParameter>
           					<CssParameter name="font-style">normal</CssParameter>
         				</Font>
                                              <LabelPlacement>
                           <PointPlacement>
                             <AnchorPoint>
                               <AnchorPointX>0.5</AnchorPointX>
                               <AnchorPointY>0.0</AnchorPointY>
                             </AnchorPoint>
                           </PointPlacement>
                         </LabelPlacement>
       				</TextSymbolizer>
				</Rule>
			</FeatureTypeStyle>
		</UserStyle>
	</NamedLayer>
</StyledLayerDescriptor>













