# Openshift CLI (`oc`)

`oc` allows you to manage your Openshift projects and applications through a terminal. Similar to how Openshift is built on top of Kubernetes with additional features, `oc` is built on top of `kubectl` but supports management of resources unique to Openshift. In order to allow for backwards compatibility with resources from a Kubernetes environment, `kubectl` is included in `oc`.

Enabling `oc` when minishift is running:  
`eval $(minishift oc-env)`  

The basic recipe for forming `oc` commands is:  

`oc <action> <object_type> <object_name_or_id>`  

Examples:  
`oc status` -> overview of project resources  
`oc get project` -> get current project  
`oc get pods` -> list pods (to watch, add `-w`)   
`oc describe pod <pod_id>` -> get detailed information about a pod  
`oc get pod <podname> -o yaml` -> get a pod's YAML   

Red Hat's documentation containing the different command options is [here](https://docs.openshift.com/enterprise/3.0/cli_reference/basic_cli_operations.html).  

## Exploring `oc`

To use `oc`, you must log in first.  

`oc login`  

> The default username/password is `developer/developer`  

First, we want to make sure we are in the correct project.
```bash
 λ oc get project
NAME               DISPLAY NAME   STATUS
cascon-oc-config                  Active
```

> To switch to a project, use `oc project <project_name>`   


Let's check the project status to get some basic information about our project and any applications within it.  
<pre>
 λ oc status
<b>In project cascon-oc-config on server https://192.168.64.2:8443</b>

svc/mongodb - 172.30.8.66:27017
  dc/mongodb deploys openshift/mongodb:3.2
    deployment #1 deployed 13 days ago - 1 pod

<b>http://nodejs-ex-cascon-oc-config.192.168.64.2.nip.io</b> to pod port 8080-tcp (svc/nodejs-ex)
  <b>dc/nodejs-ex</b> deploys istag/nodejs-ex:latest <-
    <b>bc/nodejs-ex</b> source builds <b>https://github.com/jwsliu/nodejs-ex</b> on openshift/nodejs:10
    deployment #5 deployed 13 days ago - 1 pod
    deployment #4 deployed 13 days ago
    deployment #3 deployed 13 days ago
</pre>

> Here we can see that we have a mongodb service, and that it has 1 deployment config (dc) and 1 deployment.
> We can also see a nodejs service with its route, deployment config (dc), build config (bc), and deployment history (3 deployments)

Let's check on our pods.  

```bash
 λ oc get pods -w
NAME                READY     STATUS      RESTARTS   AGE
mongodb-1-7xgzg     1/1       Running     1          13d
nodejs-ex-1-build   0/1       Completed   0          13d
nodejs-ex-5-kj5pc   1/1       Running     1          13d
```

Scale up the application in the web console to see real-time updates.  

Let's check the logs for our nodejs pod.

```bash
 λ oc logs -f nodejs-ex-5-kj5pc
git version 1.8.3.1
Environment:
	DEV_MODE=false
	NODE_ENV=production
	DEBUG_PORT=5858
Running as user uid=1001(default) gid=0(root) groups=0(root)
Launching via npm...
npm info it worked if it ends with ok
npm info using npm@6.9.0
npm info using node@v10.16.3
npm info lifecycle nodejs-ex@0.0.1~prestart: nodejs-ex@0.0.1
npm info lifecycle nodejs-ex@0.0.1~start: nodejs-ex@0.0.1
> nodejs-ex@0.0.1 start /opt/app-root/src
> node server.js
Server running on http://0.0.0.0:8080
Connected to MongoDB at: mongodb://userUAY:GiDdfPOX7B1UTGnX@172.30.8.66:27017/sampledb
172.17.0.1 - - [24/Oct/2019:16:00:49 +0000] "GET / HTTP/1.1" 200 36928 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:70.0) Gecko/20100101 Firefox/70.0"
```

> If we refresh the page, we should see a new HTTP GET request.

## Openshift Resources
1. **Deployment Config**  
Similar to a Deployment in Kubernetes in that it describes the desired state of a component of the application, but with some extra features like automatic rollbacks and lifecycle hooks  
`oc get dc`  
`oc get dc -o yaml`  
`oc describe dc <dc_id>`  
<pre>
 λ oc describe dc nodejs-ex
Name:		nodejs-ex
Namespace:	cascon-oc-config
Created:	2 weeks ago
Labels:		app=nodejs-ex
Annotations:	openshift.io/generated-by=OpenShiftNewApp
Latest Version:	7
Selector:	app=nodejs-ex,deploymentconfig=nodejs-ex
Replicas:	1
Triggers:	Config, Image(nodejs-ex@latest, auto=true)
<b>Strategy:	Rolling</b>
Template:
Pod Template:
  Labels:	app=nodejs-ex
		deploymentconfig=nodejs-ex
  Annotations:	openshift.io/generated-by=OpenShiftNewApp
  <b>Containers:</b>
   <b>nodejs-ex:</b>
    Image:	172.30.1.1:5000/cascon-oc-config/nodejs-ex@sha256:5b48b528252a0b9ae4be130eb5f5db830764096fae61f5ed0ffb6307b015e643
    Port:	8080/TCP
    Host Port:	0/TCP
    Environment:
      DATABASE_SERVICE_NAME:	mongodb
      MONGODB_USER:		<set to the key 'database-user' in secret 'mongodb'>	Optional: false
      MONGODB_DATABASE:		<set to the key 'database-name' in secret 'mongodb'>	Optional: false
      MONGODB_PASSWORD:		badpassword
      MONGODB_ADMIN_PASSWORD:	<set to the key 'database-admin-password' in secret 'mongodb'>	Optional: false
    Mounts:			<none>
  Volumes:			<none>

<b>Deployment #7 (latest):</b>
	Name:		nodejs-ex-7
	Created:	about an hour ago
	Status:		Complete
	Replicas:	1 current / 1 desired
	Selector:	app=nodejs-ex,deployment=nodejs-ex-7,deploymentconfig=nodejs-ex
	Labels:		app=nodejs-ex,openshift.io/deployment-config.name=nodejs-ex
	Pods Status:	1 Running / 0 Waiting / 0 Succeeded / 0 Failed
Deployment #6:
	Created:	about an hour ago
	Status:		Failed
	Replicas:	0 current / 0 desired
Deployment #5:
	Created:	13 days ago
	Status:		Complete
	Replicas:	0 current / 0 desired

Events:
  Type		Reason				Age			From				Message
  ----		------				----			----				-------
<b>  Normal	DeploymentCreated		59m			deploymentconfig-controller	Created new replication controller "nodejs-ex-7" for version 7</b>
  Normal	RolloutCancelled		59m			deployer-controller		Rollout for "cascon-oc-config/nodejs-ex-6" cancelled
  Normal	DeploymentAwaitingCancellation	59m (x7 over 59m)	deploymentconfig-controller	Deployment of version 7 awaiting cancellation of older running deployments
  Normal	DeploymentCancelled		59m			deploymentconfig-controller	Cancelled deployment "nodejs-ex-6" superceded by version 7
</pre>
> [Understanding Deployments and DeploymentConfigs - Deployments | Applications | OpenShift Container Platform 4.1](https://docs.openshift.com/container-platform/4.1/applications/deployments/what-deployments-are.html)  

2. **Build Config**  
A build config describes how to take input like source code or an image and create a build, and ties in build hooks and triggers  
`oc get bc`  
`oc get bc -o yaml`  
`oc describe bc <bc_id>`    
<pre>
 λ oc describe bc nodejs-ex
Name:		nodejs-ex
Namespace:	cascon-oc-config
Created:	2 weeks ago
Labels:		app=nodejs-ex
Annotations:	openshift.io/generated-by=OpenShiftNewApp
Latest Version:	1

<b>Strategy:	Source</b>
<b>URL:		https://github.com/jwsliu/nodejs-ex</b>
From Image:	ImageStreamTag openshift/nodejs:10
<b>Output to:	ImageStreamTag nodejs-ex:latest</b>

Build Run Policy:	Serial
Triggered by:		Config, ImageChange
Webhook GitHub:
	URL:	https://192.168.64.2:8443/apis/build.openshift.io/v1/namespaces/cascon-oc-config/buildconfigs/nodejs-ex/webhooks/<secret>/github
Webhook Generic:
	URL:		https://192.168.64.2:8443/apis/build.openshift.io/v1/namespaces/cascon-oc-config/buildconfigs/nodejs-ex/webhooks/<secret>/generic
	AllowEnv:	false
Builds History Limit:
	Successful:	5
	Failed:		5

Build		Status		Duration	Creation Time
nodejs-ex-1 	complete 	18s 		2019-10-10 14:06:17 -0400 EDT

Events:	<none>
</pre>
> [How Builds Work - Builds | Developer Guide | OpenShift Container Platform 3.4](https://docs.openshift.com/container-platform/3.4/dev_guide/builds/index.html)  

3. **Image Streams**  
Image streams are a way of abstracting references to Docker images, using tags to point to specific container images  

`oc get is`  
`oc describe is <is_id>`   

Example: Using ImageStreams  

`oc get is` -> list image streams  
`oc tag nodejs-ex:latest nodejs-ex:test5` -> tag latest build as "test5"

```bash
 λ oc describe is nodejs-ex
Name:			nodejs-ex
Namespace:		cascon-oc-config
Created:		2 weeks ago
Labels:			app=nodejs-ex
Annotations:		openshift.io/generated-by=OpenShiftNewApp
Docker Pull Spec:	172.30.1.1:5000/cascon-oc-config/nodejs-ex
Image Lookup:		local=false
Unique Images:		1
Tags:			3

latest
  no spec tag

  * 172.30.1.1:5000/cascon-oc-config/nodejs-ex@sha256:5b48b528252a0b9ae4be130eb5f5db830764096fae61f5ed0ffb6307b015e643
      2 weeks ago

test1
  tagged from nodejs-ex@sha256:5b48b528252a0b9ae4be130eb5f5db830764096fae61f5ed0ffb6307b015e643

  * 172.30.1.1:5000/cascon-oc-config/nodejs-ex@sha256:5b48b528252a0b9ae4be130eb5f5db830764096fae61f5ed0ffb6307b015e643
      13 days ago

test5
  tagged from nodejs-ex@sha256:5b48b528252a0b9ae4be130eb5f5db830764096fae61f5ed0ffb6307b015e643

  * 172.30.1.1:5000/cascon-oc-config/nodejs-ex@sha256:5b48b528252a0b9ae4be130eb5f5db830764096fae61f5ed0ffb6307b015e643
      11 seconds ago
```
> [How to Simplify Container Image Management in Kubernetes with OpenShift Image Streams – Red Hat OpenShift Blog](https://blog.openshift.com/image-streams-faq/)  

# Breaking Stuff

Let's break our app! 

1. Make sure everything is working  
```bash
 λ oc get pods
NAME                READY     STATUS      RESTARTS   AGE
mongodb-1-7xgzg     1/1       Running     1          13d
nodejs-ex-1-build   0/1       Completed   0          13d
nodejs-ex-7-kj5pc   1/1       Running     0          35s
```
2. Mess up a deployment config

```bash
λ oc edit dc nodejs-ex
 ```

Look for this part of the deployment config:
 ```yaml
 spec:
  containers:
  - env:
    - name: MONGODB_USER
      valueFrom:
        secretKeyRef:
          key: database-user
          name: mongodb
    - name: MONGODB_PASSWORD
      valueFrom:
        secretKeyRef:
          key: database-password
          name: mongodb
    - name: MONGODB_ADMIN_PASSWORD
      valueFrom:
        secretKeyRef:
          key: database-admin-password
          name: mongodb
    - name: MONGODB_DATABASE
      valueFrom:
        secretKeyRef:
          key: database-name
          name: mongodb
```

Change the password to a regular string:
```yaml
    - name: MONGODB_PASSWORD
      value: wrongpassword
```

Save the file, and Openshift will use the new deployment config to create a new pod where the nodejs application can no longer connect to the database.

If you broke it correctly, the new pod should be up and running.

```bash
 λ oc get pods
NAME                READY     STATUS      RESTARTS   AGE
mongodb-1-7xgzg     1/1       Running     1          13d
nodejs-ex-1-build   0/1       Completed   0          13d
nodejs-ex-7-tgjj2   1/1       Running     0          34m
```

Check the logs for the new pod

```bash
 λ oc logs nodejs-ex-7-tgjj2
git version 1.8.3.1
Environment:
	DEV_MODE=false
	NODE_ENV=production
	DEBUG_PORT=5858
Running as user uid=1001(default) gid=0(root) groups=0(root)
...
Server running on http://0.0.0.0:8080
Error connecting to Mongo. Message:
MongoError: Authentication failed.
```

Check the application, if you're unsure of the url, check the routes:

```bash
 λ oc get routes
NAME        HOST/PORT                                        PATH      SERVICES    PORT       TERMINATION   WILDCARD
nodejs-ex   nodejs-ex-cascon-oc-config.192.168.64.2.nip.io             nodejs-ex   8080-tcp                 None
```

The application now says that there is no database configured.  
<img src="docImages/nodb-configured.png" alt="No database configured" title="No database configured" height="75" />

# ODO

`odo` (or Openshift Do) is essentially a wrapper around the `oc` command-line tool to simplify everything for developers who might not know the details about Kubernetes or Openshift

In 3 commands you can build and deploy a new application with just the source code.  
`odo create nodejs`  
`odo url create`  
`odo push`  

Other `odo` commands:  
`odo project create myproject`  
`odo catalog list components`  
`odo log -f`  
`odo create nodejs frontend`  
`odo url create` or `odo url create frontend --port 8080`  
`odo push`  
`odo watch` or `odo watch &`  
`odo url list`  
