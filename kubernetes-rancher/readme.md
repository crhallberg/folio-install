# folio-install on Kubernetes/Rancher 2.0

## License

Copyright (C) 2016-2018 The Open Library Foundation

This software is distributed under the terms of the Apache License, Version 2.0. See the file "[LICENSE](LICENSE)" for more information.

## Introduction

A collection of containers and YAML files for FOLIO installation on Kubernetes/Rancher 2.0.

## Contents

* [Minikube deployment notes](Folio_MiniKube_Notes.md)



## Prerequisites for Rancher Server ##



### In Oralce Linux, install correct version of Docker:

yum install yum-utils

yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

yum install --setopt=obsoletes=0 \
   docker-ce-17.03.2.ce-1.el7.centos.x86_64

systemctl start docker.service && systemctl enable docker.service

### Add insecure Docker Registry:

cat > /etc/docker/daemon.json <<END
{
  "insecure-registries" : ["<Your Docker private registry IP or FQDN>"],
  "storage-driver": "devicemapper"
}
END

### Restart Docker after ammending Docker Registry:

systemctl restart docker

### Run/Install Rancher Manager:

docker run -d --restart=unless-stopped -p 80:80 -p 443:443 rancher/rancher



## Prerequisites for Kubernetes Hosts ##



### In Oralce Linux, install correct version of Docker:

yum install yum-utils

yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

yum install --setopt=obsoletes=0 \
   docker-ce-17.03.2.ce-1.el7.centos.x86_64

systemctl start docker.service && systemctl enable docker.service

### Add insecure Docker Registry:

cat > /etc/docker/daemon.json <<END
{
  "insecure-registries" : ["<Your Docker private registry IP or FQDN>"],
  "storage-driver": "devicemapper"
}
END

### Restart Docker after ammending Docker Registry:

systemctl restart docker

### Setup Kubernetes Cluster (From within Rancher GUI):

1) Add a Cluster -> select custom -> enter a cluster name (Folio) next
2) From Node Role, select all the roles: etcd, Control, and Worker.
3) Optional: Rancher auto-detects the IP addresses used for Rancher communication and cluster communication.
   You can override these using Public Address and Internal Address in the Node Address section.
4) Skip the Labels stuff. It's not important for now.
5) Copy the command displayed on screen to your clipboard.
6) Log in to your first Kubernetes Linux host using your preferred shell, such as PuTTy or a remote Terminal connection. Run the command copied to your clipboard. Repeat this for each additional node you wish to add.
7) When you finish running the command on all of your Kubernetes Linux hosts, click Done inside of Rancher GUI.



## FOLIO in Kubernetes Notes ##



### Set-up order overview (Our Rancher-exported YAML can be looked at under the YAML folder):
•	Create cluster via Rancher 2.0 with one or more nodes, using Canal network plugin. We are using VMware to provision Oracle Linux VMs at the moment
•	Create Folio-Project in Rancher 2.0 UI
•	Add folio-q2 Namespace for Folio-Project under Namespaces in Rancher 2.0 UI
•	Add Dockerhub and your private Docker registries to the Folio-Project
•	Add Persistent Volume on the cluster and Persistent Volume Claim for Folio-Project (We are using an NFS Share)
•	(The rest of these steps are from within the Folio-Project in Rancher 2.0)
•	Deploy crunchy-postgres Workload 'Stateful set' via Helm Package, edit the Workload to tweak environment variables for use with Folio
•	Add Record under Service Discovery, named pg-folio, as type 'Selector' with Label/Value: statefulset.kubernetes.io/pod-name = pgset-0
•	Deploy create-db Workload ‘Job’ - our custom Docker container with DB init scripts
•	Deploy Okapi Workload 'Scalable deployment' of 1 - our custom Docker container
•	Deploy Folio module Workloads 'Scalable deployment' of 3 (one Workload per Folio module)
•	Deploy Stripes Workload 'Run one pod on each node' – our custom Docker container
•	Deploy create-tenant/create-deploy Workload ‘Job’ – our custom Docker container with scripts
•	Edit Okapi Workload, set InitDB environment variable to false
•	Scale up Okapi pods to 3 using Rancher 2.0 + button
•	Add Ingress under Load Balancing for Okapi and Stripes using URLs for `/` and `/_/`

### Okapi Notes:
•	Running in "clustered mode", but is not currently clustering
•	Workload running as 3 pods that share database
•	Initially spin up one Okapi pod, do the deployment jobs, then can scale out Okapi's pods and they will each pick up the tenants/discovery/proxy services
•	After single Okapi pod has been initialized, set Workload environment variable for InitDB to false for future pod scalability
•	Running Okapi with 'ClusterIP' type of port mapping, and Clusterhost IP environment variable set to the assigned 'ClusterIP' of the service as given by Kubernetes/Rancher

### Okapi Workload environment variables:
	
PG_HOST					pg-folio
OKAPI_URL				http://okapi:9130
OKAPI_PORT				9130
OKAPI_NODENAME			okapi1
OKAPI_LOGLEVEL			INFO
OKAPI_HOST				okapi
OKAPI_CLUSTERHOST		xx.xx.x.xxx (Insert ClusterIP Kubernetes assigns the service)
INITDB					false
HAZELCAST_VERTX_PORT	5703
HAZELCAST_PORT			5701
HAZELCAST_IP			xx.xx.x.xxx (Insert ClusterIP Kubernetes assigns the service)


### HA Postgres in Kubernetes/Rancher Notes:
•	Currently testing out crunchy-postgres HA Kubernetes solution
•	Running as a Kubernetes 'Stateful Set', with one primary and two replica pods. Replica pods are read-only
•	For Postgres 'Service Discovery', created a 'Selector' type called ‘pg-folio’ that targets the master pgset-0 Postgres pod via a label
•	Using a 'Persistent Volume Claim' for Rancher Folio Project, which is a 10GB NFS share on our Netapp filer
•	Not sure if we would run like this in Production yet, as we haven't load tested it. It is a possibility for those looking for a complete Kubernetes/Container solution and being actively developed out more

### Crunchy-postgres Workload environment variables:

WORK_MEM				32
PGHOST					/tmp
PGDATA_PATH_OVERRIDE	folio-data
PG_USER					okapi
PG_ROOT_PASSWORD		<Postgres user DB password>
PG_REPLICA_HOST			pgset-replica
PG_PRIMARY_USER			primaryuser
PG_PRIMARY_PORT			5432
PG_PRIMARY_PASSWORD		<Primaryuser of the set DB password>
PG_PRIMARY_HOST			pgset-primary
PG_PASSWORD				okapi25
PG_MODE					set
PG_LOCALE				en_US.UTF-8
PG_DATABASE				okapi
MAX_CONNECTIONS			150
ARCHIVE_MODE			off

### Modules that require a database Workload environment variables:

db.username 			folio_admin
db.port 				5432
db.password 			folio_admin
db.host 				pg-folio
db.database 			okapi_modules

### mod-authtoken Workload environment variable:

JAVA_OPTIONS			-Djwt.signing.key=CorrectBatteryHorseStaple


### Ingress Notes:
•	Have two URLs as A Records for the three Kube nodes
•	One URL is for proxying front-end and the other is for proxying Okapi traffic
•	The Okapi traffic URL is the URL used when building Stripes
•	When setting up Load Balancing/Ingress, target the Service name instead of Workload name if you have specific ports you have set in the Workload


## Pro Tips ##


### Config and launch Kubernetes Cluster Dashboard from your workstation (Save files and token on iMac in /Users/UserName/.kube/):

Run instructions from here: https://gist.github.com/superseb/3a9c0d2e4a60afa3689badb1297e2a44

### Build, tag and push docker images from within their respective folders:

docker build -t <my-docker-private-registry>/folio-project/containername:v1 .
docker push <my-docker-private-registry>/folio-project/containername:v1

### To clean existing Kubernetes cluster node for re-deployment run:

docker stop $(docker ps -aq)
docker rm $(docker ps -aq)
docker rmi $(docker images -q)
docker volume prune
docker system prune
sudo rm -rf /var/lib/etcd /etc/kubernetes/ssl /etc/cni /opt/cni /var/lib/cni /var/run/calico /etc/kubernetes/.tmp/
systemctl restart docker

### To remove dangling images from nodes that have failed:

docker rmi $(docker images -aq --filter dangling=true)

### When spinning up containers from Dockerhub, be sure the Workload `Advanced Option - Command` "Auto Restart" setting is set to "Always"

### When creating Workloads, set min to 0 if you wish to scale your deployments to 0 - in Kubernetes Dashboard

### When using crunchy-postgres, need to execute this command on the Kubernetes cluster for the pgset-sa service account:

kubectl create clusterrolebinding pgset-sa --clusterrole=admin --serviceaccount=folio-q2:pgset-sa --namespace=folio-q2

### To find out which orphanded Replicasets need deleting that have 0 pods (Run this in Rancher using Launch kubectl):

kubectl get --all-namespaces rs -o json|jq -r '.items[] | select(.spec.replicas | contains(0)) | "kubectl delete rs --namespace=\(.metadata.namespace) \(.metadata.name)"'

### Set .spec.revisionHistoryLimit in the Kubernetes Deployment to the number of old Replicasets you want to retain.
