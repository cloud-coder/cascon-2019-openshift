# Templates
You can find the templates guide here: [Templates](https://docs.openshift.com/online/dev_guide/templates.html)

Templates are useful for scenarios when you are creating a set of objects that share some common values, which can be parameterized. Or you want to apply the same label across multiple objects. A template definition has three main sections: **Labels**, **Parameters** and **Objects**. The **Objects** section contains an array of resource definitions that you would like to create. Certain values in the resource definitions have a placeholder (i.e. variable, parameter) for a value. The name of this placeholder/variable is defined in the **Parameters** section, and to insert the value of the parameter, you would use *${param-name}* notation. A template itself is a resource definition like any other OpenShift (or Kubernetes) resources.

Here is an example of a template from the OpenShift [documentation](https://docs.openshift.com/online/dev_guide/templates.html#writing-templates):
```yaml
apiVersion: v1
kind: Template
metadata:
  name: redis-template
  annotations:
    description: "Description"
    iconClass: "icon-redis"
    tags: "database,nosql"
objects:
- apiVersion: v1
  kind: Pod
  metadata:
    name: redis-master
  spec:
    containers:
    - env:
      - name: REDIS_PASSWORD
        value: ${REDIS_PASSWORD}
      image: dockerfile/redis
      name: master
      ports:
      - containerPort: 6379
        protocol: TCP
parameters:
- description: Password used for Redis authentication
  from: '[A-Z0-9]{8}'
  generate: expression
  name: REDIS_PASSWORD
labels:
  redis: master
```

Before a template can be used to create the resources defined in it, you would need to process it first. This simply replaces the placeholders with the actual values in the resources. These parameters can be overridden when run from the command line, or they can be randomly generated at processing time. The command to process a template is `oc process` which you can either pass it an existing template that is already defined in OpenShift, or you can can pass a template file using `-f filename` option. You can view the output of the processing as it is printed out in the command line, or you can pipe it to a file or a `oc create` command.

# Writing a Template
Starting from scratch to write a template file is generally hard to do. Instead of writing the template manually from scratch, we could either look at existing templates that are closer to what we want to create or create the resources using other commands and methods, then export and use the generated resource definitions.

## Using Existing Templates
A typical installation of OpenShift comes with a number of templates. These templates, which can be found in the catalog, are good sources for examples and learning. Let's list these templates and review one of them as an example.

These templates are generally defined in the `openshift` namespace, which makes them accessible to the rest of the projects. The command `oc get templates -n openshift` lists these templates. Examine the list and you will find one template that is called `nodejs-mongo-persistent`.

So for example you can run `oc get templates -n openshift | grep -i -E "node|NAME"` command and you will see the following results

```
NAME                                            DESCRIPTION                                                                        PARAMETERS        OBJECTS
nodejs-mongo-persistent                         An example Node.js application with a MongoDB database. For more information...    19 (4 blank)      9
```

To look at the contents of the template, you could use the `oc get template nodejs-mongo-persistent --export -o yaml -n openshift` command, and you will see a result like below (it's truncated for brevity)

```yaml
apiVersion: template.openshift.io/v1
kind: Template
labels:
  template: nodejs-mongo-persistent

...

objects:
- apiVersion: v1
  kind: Secret
  metadata:
    name: ${NAME}
  stringData:
    database-admin-password: ${DATABASE_ADMIN_PASSWORD}
    database-password: ${DATABASE_PASSWORD}
    database-user: ${DATABASE_USER}
- apiVersion: v1
  kind: Service
  metadata:
    annotations:
      description: Exposes and load balances the application pods
      service.alpha.openshift.io/dependencies: '[{"name": "${DATABASE_SERVICE_NAME}",
        "kind": "Service"}]'
    name: ${NAME}
  spec:
    ports:
    - name: web
      port: 8080
      targetPort: 8080
    selector:
      name: ${NAME}

...

- apiVersion: v1
  kind: DeploymentConfig
  metadata:
    annotations:
      description: Defines how to deploy the application server
      template.alpha.openshift.io/wait-for-ready: "true"
    name: ${NAME}
  spec:
    replicas: 1
    selector:
      name: ${NAME}
    strategy:
      type: Recreate
    template:
      metadata:
        labels:
          name: ${NAME}
        name: ${NAME}
      spec:
        containers:
        - env:
          - name: DATABASE_SERVICE_NAME
            value: ${DATABASE_SERVICE_NAME}
          - name: MONGODB_USER
            valueFrom:
              secretKeyRef:
                key: database-user
                name: ${NAME}
          - name: MONGODB_PASSWORD
            valueFrom:
              secretKeyRef:
                key: database-password
                name: ${NAME}
          - name: MONGODB_DATABASE
            value: ${DATABASE_NAME}
          - name: MONGODB_ADMIN_PASSWORD
            valueFrom:
              secretKeyRef:
                key: database-admin-password
                name: ${NAME}
          image: ' '
          livenessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 30
            timeoutSeconds: 3
          name: nodejs-mongo-persistent
          ports:
          - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 3
            timeoutSeconds: 3
          resources:
            limits:
              memory: ${MEMORY_LIMIT}
    triggers:
    - imageChangeParams:
        automatic: true
        containerNames:
        - nodejs-mongo-persistent
        from:
          kind: ImageStreamTag
          name: ${NAME}:latest
      type: ImageChange
    - type: ConfigChange
- apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: ${DATABASE_SERVICE_NAME}
  spec:
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: ${VOLUME_CAPACITY}

...

parameters:
- description: The name assigned to all of the frontend objects defined in this template.
  displayName: Name
  name: NAME
  required: true
  value: nodejs-mongo-persistent
- description: The OpenShift Namespace where the ImageStream resides.
  displayName: Namespace
  name: NAMESPACE
  required: true
  value: openshift

...

- description: Password for the database admin user.
  displayName: Database Administrator Password
  from: '[a-zA-Z0-9]{16}'
  generate: expression
  name: DATABASE_ADMIN_PASSWORD
- description: The custom NPM mirror URL
  displayName: Custom NPM Mirror URL
  name: NPM_MIRROR

```

You save the output of this command to a file, modify as necessary and build your own template file from it.

## Using a Deployed Application
Another option is that you can deploy an application using a variety of techniques (e.g. S2I, Dockerfile,...), mixing deployment using console and then augmenting configuration using command line... And ultimately you could take the resulting resources and build your own template that would make these tasks repeatable in a consistent fashion and in a single step.

Let's a take sample scenario like below:
> We use S2I to deploy our application in the *Dev* project, which creates an *image stream* with a *latest* tag. Every time we tag an image as *test*, we want that image to be (re)deployed to the *Test* project.

Along the way, in this example, we will also look at using *Deploy Keys* to access a private GitHub repository, and also enable access across projects to get an image available in another project from the registry. And we then generate a template that can presumably be modified/parameterized for environment related configuration.

> **Note:** The build and deployment automation approach shown here may not necessarily be a best practice, but an example to demonstrate the capability

### Enable Access to Private Repository
There are several documented approaches for enabling access to private GitHub repositories. This [blog](https://blog.openshift.com/private-git-repositories-part-1-best-practices/) entry has some details. For this exercise we will use the *Deploy Keys* approach. In this approach, you would generate a public/private ssh key pair, add the public key to the *Deploy Keys* of the private repository that you would like to give access to S2I to read from. Using this approach, you can have create or share as many ssh keys as necessary and assign them to the repositories, and limit access to read only.

First generate the ssh key pair. You can follow the GitHub documentation to run the command below. Make sure that you don't assign a password (that's what the `-N` option provides).

`ssh-keygen -t rsa -b 4096 -C "my_email@mycompany.com" -N "" -f my-app-deploy-key`

Now add the **public key** to your repository's *Deploy Keys* section, as shown in this [document](https://developer.github.com/v3/guides/managing-deploy-keys/). The summary of steps to take are:
- Go to your repository's page
- Click on the *Settings* link in the same row where you see *Code* as the first option, and then *Issues*...
- Click on the *Deploy Keys* option on the left hand navigation items
- Press the *Add deploy key* button on the top right
- Add the public key as a new item with a name, and grant it read only access


### Deploy the Application
In the steps below, we will assume a Mac development machine, and paths and file names as shown below. Adjust these values per your OS and/or paths
- Project name: cascon-oc-tmpl-dev
- Application name: cascon-oc
- Secret for SSH: cascon-oc-tmpl-dev

Log in as a developer

`oc login -u developer -p developer`

Create a project to deploy the development instance of the application. This instance would for example update at every commit to the development branch.

`oc new-project cascon-oc-tmpl-dev`

Earlier we created an entry in the *Deploy Keys* section of the repository. We will use the private key to create a secret in the project so we can access the source code.

Create a secret based on the private key, in this case we'll call it *cascon-oc-tmpl-dev*, note that the type is *ssh-auth*

`oc create secret generic cascon-oc-tmpl-dev --from-file=ssh-privatekey=/Users/my_os_user/.ssh/cascon_git_id_rsa --type=kubernetes.io/ssh-auth`

Annotate the type by adding the repository URL. You can also use wildcards (refer to the article referenced earlier).

`oc annotate secret/cascon-oc-tmpl-dev 'build.openshift.io/source-secret-match-uri-1=ssh://github.ibm.com/my_git_user/cascon-2019-openshift.git'`

Link the secret to the builder service account

`oc secrets link builder cascon-oc-tmpl-dev`

Now that the secret based on the ssh key is set up, we can deploy the application using the source code (i.e. S2I)

`oc new-app ssh://git@github.ibm.com/my_git_user/cascon-2019-openshift.git --name cascon-oc`

> **Note** The url specified for the secret is different from the one specified for the application deployment

After the command returns, we can check the resources that are created and in progress to completion

`oc get all`

We can tail the build logs and see the progress of the build and image creation

`oc logs bc/cascon-oc -f`

Now the application is deployed successfully and is running, we can create a route and expose it

`oc expose svc/cascon-oc`

Verify that the route was created and view its details

`oc get routes`

Verify that the application is deployed successfully and can be accessed through the route

> **Note** Depending on your OpenShift installation, the IP address might be different, but the segment before the IP is made up application name - project name

`curl http://cascon-oc-cascon-oc-tmpl-dev.192.168.64.9.nip.io | less`



### Examine the generated resources
The above deployment created several resources. Let's explore the created *image stream* and see that it is referenced in the *DeploymentConfig*

`oc get is`

Examine the image stream in detail

`oc describe is cascon-oc`

Looking the results from the above command, we can see that there are no tags defined. One of the ways to promote a deployment is through tags. After several iterations in the *dev* project/namespace, when a particular code version needs is ready for testing, we can tag that image as test. This can be referenced from another *DeploymentConfig* in a test project, and trigger the redeployment of that particular image, and make it available for testing. This is one approach to control when to promote to test, by simply tagging the relevant image.

Let's tag the *latest* image as *test*

`oc tag cascon-oc:latest cascon-oc:test`

Re-examine the image stream and verify that the tag is created

`oc describe is cascon-oc`

### Export Resources
Assuming that we have completed the necessary configuration and the application is deployed and running successfully, we can now export the resources. In this example, we will export the *DeploymentConfig* resource and use that for next deployment.

Export the *DeploymentConfig* resource and save it into a file

`oc get dc cascon-oc --export -o yaml > cascon-oc-dc-dev.yaml`

Make a copy of the file and modify its contents to include configuration changes for a test environment deployment

`cp cascon-oc-dc-dev.yaml cascon-oc-dc-test.yaml`

The Changes are listed below

- Add a description to the deployment config
```yaml
apiVersion: apps.openshift.io/v1
kind: DeploymentConfig
metadata:
  annotations:
    description: CASCON Sample App deployment config
  generation: 1
  ...
```

Optionally remove the following lines
``` yaml
...
selfLink: /apis/apps.openshift.io/v1/namespaces/cascon-oc-tmpl-dev/deploymentconfigs/cascon-oc
...
revisionHistoryLimit: 10
...
resources: {}
...
terminationMessagePath: /dev/termination-log
terminationMessagePolicy: File
dnsPolicy: ClusterFirst
...
schedulerName: default-scheduler
securityContext: {}
terminationGracePeriodSeconds: 30
...
status:
  availableReplicas: 0
  latestVersion: 0
  observedGeneration: 0
  replicas: 0
  unavailableReplicas: 0
  updatedReplicas: 0
```

Add a description to the pod specification
```yaml
template:
    metadata:
      annotations:
        description: CASCON Sample App pod
      labels:
```

In the same pod section, there is a reference to the image with its SHA id, which makes it a specific reference to a single image. However if we want to deploy based on a tag (as described earlier) we would want to change this reference. In this case, let's replace the SHA id to a tag called *test* which we created earlier.

Changed from
```yaml
spec:
      containers:
      - image: 172.30.1.1:5000/cascon-oc-tmpl-dev/cascon-oc@sha256:4a5f6c6a9df29db320b1c4a252b7e337af7992d989d1636daa56320ce9437859
        imagePullPolicy: Always
```
to the below

```yaml
spec:
      containers:
      - image: 172.30.1.1:5000/cascon-oc-tmpl-dev/cascon-oc:test
        imagePullPolicy: Always
```

In the *imageChangeParams:* entry, change the reference from *latest* to *test*

From
```yaml
name: cascon-oc:latest
```

To
```yaml
name: cascon-oc:latest
```

The updated *DeploymentConfig* is ready to be deployed. Here is the full document after the updates
```yaml
apiVersion: apps.openshift.io/v1
kind: DeploymentConfig
metadata:
  annotations:
    description: CASCON Sample App deployment config
  generation: 1
  labels:
    app: cascon-oc
  name: cascon-oc
spec:
  replicas: 1
  selector:
    app: cascon-oc
    deploymentconfig: cascon-oc
  strategy:
    activeDeadlineSeconds: 21600
    rollingParams:
      intervalSeconds: 1
      maxSurge: 25%
      maxUnavailable: 25%
      timeoutSeconds: 600
      updatePeriodSeconds: 1
    type: Rolling
  template:
    metadata:
      annotations:
        description: CASCON Sample App pod
      labels:
        app: cascon-oc
        deploymentconfig: cascon-oc
    spec:
      containers:
      - image: 172.30.1.1:5000/cascon-oc-tmpl-dev/cascon-oc:test
        imagePullPolicy: Always
        name: cascon-oc
        ports:
        - containerPort: 8080
          protocol: TCP
        resources: {}
      restartPolicy: Always
  triggers:
  - type: ConfigChange
  - imageChangeParams:
      automatic: true
      containerNames:
      - cascon-oc
      from:
        kind: ImageStreamTag
        name: cascon-oc:test
        namespace: cascon-oc-tmpl-dev
    type: ImageChange
```

### Deploy to test project
Create the test project

`oc new-project cascon-oc-tmpl-test`

If we deploy like without any further changes, the deployment will fail. Let's see what happens

`oc create -f cascon-oc-dc-test.yaml`

Verify that it failed

`oc get all`
```
	NAME                     READY     STATUS         RESTARTS   AGE
	pod/cascon-oc-1-deploy   1/1       Running        0          5s
	pod/cascon-oc-1-j8jqd    0/1       ErrImagePull   0          4s

	NAME                                DESIRED   CURRENT   READY     AGE
	replicationcontroller/cascon-oc-1   1         1         0         5s

	NAME                                           REVISION   DESIRED   CURRENT   TRIGGERED BY
	deploymentconfig.apps.openshift.io/cascon-oc   1          1         1         config,image(cascon-oc:latest)
```

The reason for the failure is that the deployment config in the *test* project is trying to access an *image stream* from the *dev* project. By default this access is not available.

Clean up the failure by deleting all the generated resources

`oc delete all -l app=cascon-oc`

Verify the items are all deleted

`oc get all`

Using sysadmin add policy to pull from an image stream in the *dev* project from the *test* project

Log in as system admin

`oc login -u system:admin`

Update the policy

`oc policy add-role-to-group system:image-puller system:serviceaccounts:cascon-oc-tmpl-test -n cascon-oc-tmpl-dev`

Log back in as *developer*

`oc login -u developer -p developer`

Try redeploying the image

`oc create -f cascon-oc-dc-test.yaml`

Verify all the resources are created

`oc get all`

The deployment should be completed and working

### Converting to a template
Now that we have verified that the *cascon-oc-dc-test.yaml* works correctly, we could take another step and convert this to a template. For instance we could parameterize the tag and provide it at deployment time, one could be for *test* and another could be for *prod*. You could also parameterize other environment related concerns in a single template.

This is an example template based on the above *DeploymentConfig*
```yaml
apiVersion: template.openshift.io/v1
kind: Template
labels:
  template: CASCON Sample App Template
message: |-
  Application ${NAME} is created
metadata:
  annotations:
    description: An example Node.js application used for CASCON OpenShift intro session.
    iconClass: icon-nodejs
    openshift.io/display-name: CASCON Node.js sample template
    openshift.io/documentation-url: https://github.com/sclorg/nodejs-ex
    openshift.io/long-description: This template defines resources needed to develop
      a simple Node.js application.
    openshift.io/provider-display-name: CASCON
    openshift.io/support-url: https://cascon.ca
    tags: cascon,nodejs
    template.openshift.io/bindable: "false"
  name: nodejs-cascon
  namespace: openshift
objects:
- apiVersion: apps.openshift.io/v1
  kind: DeploymentConfig
  metadata:
    annotations:
      description: CASCON Sample App deployment config
    generation: 1
    labels:
      app: ${NAME}
    name: ${NAME}
  spec:
    replicas: 1
    selector:
      app: ${NAME}
      deploymentconfig: ${NAME}
    strategy:
      activeDeadlineSeconds: 21600
      rollingParams:
        intervalSeconds: 1
        maxSurge: 25%
        maxUnavailable: 25%
        timeoutSeconds: 600
        updatePeriodSeconds: 1
      type: Rolling
    template:
      metadata:
        annotations:
          description: CASCON Sample App pod
        creationTimestamp: null
        labels:
          app: ${NAME}
          deploymentconfig: ${NAME}
      spec:
        containers:
        - image: 172.30.1.1:5000/cascon-oc-tmpl-dev/cascon-oc:test
          imagePullPolicy: Always
          name: ${NAME}
          ports:
          - containerPort: 8080
            protocol: TCP
          resources: {}
        restartPolicy: Always
    triggers:
    - type: ConfigChange
    - imageChangeParams:
        automatic: true
        containerNames:
        - cascon-oc
        from:
          kind: ImageStreamTag
          name: ${NAME}:test
          namespace: cascon-oc-tmpl-dev
      type: ImageChange
parameters:
- description: The name assigned to the application
  displayName: Name
  name: NAME
  required: true
  value: cascon-oc
```

In this example, there is only one attribute that is parameterized and that is the name. This notion can be extended, based on application needs, and further parameterize other aspects.

### Processing the template
After creating the template file show above (e.g. *cascon-oc-tmpl.template.yaml*), we can process it with the following command

`oc process -f cascon-oc-tmpl.template.yaml | less`

We can either direct the output of the processing to a file or to an `oc new-app` command in one go, to create the resources. It is also possible to create the template in the *openshift* project/namespace and allow others to use the same template to create a similar application.
