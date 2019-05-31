> CLI-notes.md

# Useful Generic Commands

* CLI Help

      oc help

* List all the object types (and definitions!)

      oc types

* Get detailed help on an object type

      oc explain <objecttype>

* Login to a cluster

      oc login <cluster>

* Logoff from a cluster

      oc logout

* Get a list of objects

      oc get <objecttype>

* Get details on an object

      oc describe <object>

* Edit object configuration

      oc edit <objecttype> <object>

* Delete an object
   
      oc delete <objecttype> <object>

* Return session information

      oc whoami

# Managing Users

* List all users

       oc get user

* List all identities

       oc get identity

* List all groups

       oc get groups

* List all service accounts

       oc get sa

* Create a new group

       oc adm groups new <group_name> <user1> <user2>

* Add user using htpasswd

       htpasswd -b /etc/origin/master/htpasswd rob.brucks mypassword

* Grant cluster admin privilege

       oc adm policy add-cluster-role-to-user cluster-admin rob.brucks

* Revoke cluster admin privilege

       oc adm policy remove-cluster-role-from-user cluster-admin rob.brucks

* See available cluster-level roles

       oc get clusterroles

* List all role bindings in a project

       oc get rolebindings

* List all role bindings for the cluster

       oc get clusterrolebindings

* Remove roles from groups

       oc adm policy remove-cluster-role-from-group self-provisioner \
          system:authenticated system:authenticated:oauth

* Grant role

       oc adm policy add-role-to-user edit rob.brucks -n mynamespace

* Revoke role

       oc adm policy remove-role-from-user edit rob.brucks -n mynamespace

* Grant project admin to user

       oc adm policy add-role-to-user admin rob.brucks -n mynamespace

* Revoke project admin from user

       oc adm policy remove-role-from-user admin rob.brucks -n mynamespace

* See all users that can perform an action

       oc adm policy who-can get pods
       oc adm policy who-can admin cluster

## Role Based Access Control (RBAC)

### Cluster RBAC

Roles and bindings that are applicable across all projects. Roles that exist cluster-wide are considered cluster roles. Cluster role
bindings can only reference cluster roles.
  
### Local RBAC

Roles and bindings that are scoped to a given project. Roles that exist only in a project are considered local roles. Local role
bindings can reference both cluster and local roles.

### Commonly Used Roles

Commonly Used Roles | Description
--- | ---
**admin** | A project manager. If used in a local binding, an admin user will have rights to view any resource in the project and modify any resource in the project except for quota.
**basic-user** | A user that can get basic information about projects and users.
**cluster-admin** | A super-user that can perform any action in any project. When bound to a user with a local binding, they have full control over quota and every action on every resource in the project.
**cluster-status** | A user that can get basic cluster status information.
**edit** | A user that can modify most objects in a project, but does not have the power to view or modify roles or bindings.
**self-provisioner** | A user that can create their own projects.
**view** | A user who cannot make any modifications, but can see most objects in a project. They cannot view or modify roles or bindings.
**cluster-reader** | A user who can read, but not view, objects in the cluster.

# Managing Projects (Namespaces)

* Create a new project

       oc new-project linuxacademy --description="ex280 class project"

* Delete a project

       oc delete project linuxacademy

* Get project overview info

       oc status

* List all projects

       oc projects

* Switch to another project

       oc project <project>


# Resource Quotas

https://docs.okd.io/latest/admin_guide/quota.html

A resource quota, defined by a ResourceQuota object, provides constraints that limit aggregate resource consumption per project. It
can limit the quantity of objects that can be created in a project by type, as well as the total amount of compute resources and
storage that may be consumed by resources in that project.

* List quotas

       oc get quota
       oc describe quota <quota>

* Create Quota for Object Counts

       apiVersion: v1
       kind: ResourceQuota
       metadata:
         name: core-object-counts
       spec:
         hard:
           configmaps: "10" 
           persistentvolumeclaims: "4" 
           replicationcontrollers: "20" 
           secrets: "10" 
           services: "10" 

* Create Quota for Resources

       apiVersion: v1
       kind: ResourceQuota
       metadata:
         name: compute-resources
       spec:
         hard:
           pods: "4" 
           requests.cpu: "1" 
           requests.memory: 1Gi 
           requests.ephemeral-storage: 2Gi 
           limits.cpu: "2" 
           limits.memory: 2Gi 
           limits.ephemeral-storage: 4Gi 

* Creating quotas from yaml files

       oc create -f <filename.yml> -n <namespace>

# Limit Ranges

https://docs.okd.io/latest/admin_guide/limits.html

A limit range, defined by a LimitRange object, enumerates compute resource constraints in a project at the pod, container, image,
image stream, and persistent volume claim level, and specifies the amount of resources that a pod, container, image, image stream,
or persistent volume claim can consume.

* List Limit Ranges

       oc get limitrange
       oc describe limitrange <quota>

* Core Limit Range

       apiVersion: "v1"
       kind: "LimitRange"
       metadata:
         name: "core-resource-limits" 
       spec:
         limits:
           - type: "Pod"
             max:
               cpu: "2" 
               memory: "1Gi" 
             min:
               cpu: "200m" 
               memory: "6Mi" 
           - type: "Container"
             max:
               cpu: "2" 
               memory: "1Gi" 
             min:
               cpu: "100m" 
               memory: "4Mi" 
             default:
               cpu: "300m" 
               memory: "200Mi" 
             defaultRequest:
               cpu: "200m" 
               memory: "100Mi" 
             maxLimitRequestRatio:
               cpu: "10" 

# Creating Openshift Apps

## Application Concepts

### Containers

The basic units of OKD applications are called containers. Linux container technologies are lightweight mechanisms for
isolating running processes so that they are limited to interacting with only their designated resources.

### Images

Containers in OKD are based on Docker-formatted container images. An image is a binary that includes all of the requirements
for running a single container, as well as metadata describing its needs and capabilities.

### Registries

A container image registry is a service for storing and retrieving Docker-formatted container images. A registry contains a
collection of one or more image repositories. Each image repository contains one or more tagged images. Docker provides its own
registry, the Docker Hub, and you can also use private or third-party registries.

### Pods

OKD leverages the Kubernetes concept of a pod, which is one or more containers deployed *together on one host*, and the smallest
compute unit that can be defined, deployed, and managed. A pod can contain just one container.

### Services

A Kubernetes service serves as an internal load balancer. It identifies a set of replicated pods in order to proxy the connections
it receives to them. Backing pods can be added to or removed from a service arbitrarily while the service remains consistently
available, enabling anything that depends on the service to refer to it at a consistent address. The default service clusterIP
addresses are from the OKD internal network and they are used to permit pods to access each other.

### Builds

A build is the process of transforming input parameters into a resulting object. Most often, the process is used to transform input
parameters or source code into a runnable image. A BuildConfig object is the definition of the entire build process.

### Image Streams

An image stream and its associated tags provide an abstraction for referencing container images from within OKD. The image stream
and its tags allow you to see what images are available and ensure that you are using the specific image you need even if the image
in the repository changes.

### Replication Controllers

A replication controller ensures that a specified number of replicas of a pod are running at all times. If pods exit or are deleted,
the replication controller acts to instantiate more up to the defined number. Likewise, if there are more running than desired, it
deletes as many as necessary to match the defined amount.

### Deployments

Building on replication controllers, OKD adds expanded support for the software development and deployment lifecycle with the
concept of deployments. In the simplest case, a deployment just creates a new replication controller and lets it start up pods.
However, OKD deployments also provide the ability to transition from an existing deployment of an image to a new one and also define
hooks to be run before or after creating the replication controller.


## Creating Applications

* Create test application

      oc new-app openshift/hello-openshift
      oc new-app centos/httpd-24-centos7~https://github.com/sclorg/httpd-ex.git

* Create new app from yaml or json files

      git clone https://github.com/linuxacademy/content-openshift-ex280.git \
          -b release-3.9
      oc create -f \
         content-openshift-ex280/origin/examples/hello-openshift/hello-pod.json

* Create a new app from github (S2I)

      oc new-app \
         centos/python-35-centos7~https://github.com/sclorg/django-ex

## Viewing objects in a project

* See objects in a project

      oc get all

* Get all pods in a project

      oc get pods
      oc get pods -o wide       # get more info
      oc get pods -n <project>  # show pods for a different project

* Get current project status

      oc status

* Check pod logs

      oc logs pod/httpd-ex-1-6mpn8
      oc logs -f pod/httpd-ex-1-6mpn8  # to "follow" the log

* Shell into a pod

      oc rsh httpd-ex-1-6mpn8

* Get deployment configs

      oc get dc

* Get deployment config details

      oc describe dc hello-openshift

* Start a new build

      oc start-build buildconfigs/django-ex

## Scaling Apps

* Scaling up pods

      oc scale --replicas=5 dc/hello-openshift

* Scaling down pods

      oc scale --replicas=1 dc/hello-openshift
   
* Scale an app to zero is perfectly valid

      oc scale --replicas=0 dc/hello-openshift

* Auto-scaling (requires cluster metrics)

      oc autoscale dc/httpd-ex --min=1 --max=5 --cpu-percent=80

## Persistent Storage

* List PVs

      oc get pv

* Describe PV

      oc describe pv <pvname>

* List PVCs

      oc get pvc

* Describe PVC

      oc describe pvc <pvcname>

* List a deployment config volume

      oc volume dc/docker-registry

Note: "empty directory" means ephemeral storage is used

* Switch from ephemeral volume to PVC volume

      oc volume dc/my-dc --add --name=my-persistent-storage \
         -t pvc --claim-name=pvc00001 --overwrite

## Exposing a service outside the project

* Expose a service

      oc expose service httpd-ex

* Create a route

      oc create route edge --service=httpd-ex --hostname=httpd-ex.classroom.apps.okd.example.com

# Managing the cluster

* Get a list of nodes and status

      oc get nodes
      oc get nodes -o wide

* Get node details

      oc describe node <nodename>

* List the pods on a node

      oc adm manage-node <nodename> --list-pods

* Evacuate and unschedule a node

      oc adm drain <nodename> --delete-local-data --force --ignore-daemonsets

* Re-schedule a node

      oc adm manage-node <nodename> --schedulable=true

* Monitor thin pool (better way?)

      journalctl -u dm-event.service -n10
      docker system df -v
      docker info
      vgs
      docker volume ls
      docker volume ls -f dangling=true
      for i in $(docker ps -q);do
        echo "Container $i"
        docker inspect -f '{{ json .Mounts }}' $i | python -m json.tool
        echo "-------------------"
      done

* Reclaim thin pool

      docker volume prune
      docker container prune
      


