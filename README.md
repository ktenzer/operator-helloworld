# Operator Framework Ansible Training
This training will show you how to setup a development environment and deploy your first Operator written in Ansible. Upon completing this training you will learn the following:
* Setup Ansible Operator Development Environment
* Create Operator Scaffolding and CRD/CR (APIs)
* Test and Debug Operators
* Read parameter inputs from CRs into Ansible facts
* Read and Create k8s objects using the k8s Ansible Module
* Deploy Application

## Prerequisites
* A fedora 30 or higher system with access to the Internet
* OpenShift 4.x environment and a cluster admin account

## Setup Development Environment
There are some prerequisites needed to develop and build an operator using Ansible. Also this guide and the operator-sdk assume you know Ansible roles. If you aren not yet up to speed please read about Ansible roles before proceeding.

### Install Docker 17.03+

Add the docker ce repositories. Note: you can also use podman and buildah instead of Docker for those that want a complete and clean divorce.

```$ sudo dnf -y install dnf-plugins-core```

```$ sudo dnf config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo```

### Install docker-ce

```$ sudo dnf -y install docker-ce docker-ce-cli```

```$ sudo systemctl start docker```

```$ sudo systemctl enable docker```

### Install Ansible and Module Dependencies

#### Install ansible

```$ sudo dnf install -y ansible```

The Ansible runner and http runner is used to run a local version of the operator. This is very useful for development and testing.

```$ pip3 install --user ansible-runner```

```$ pip3 install --user ansible-runner-http```

#### Install required python modules

```$ pip3 install --user requests``` 

```$ pip3 install --user openshift```

### Install Make utility
```$ sudo dnf install -y make```

### Install OpenShift Command Line Tools
The OpenShift command line tools ships with oc and kubectl binaries.

[Download](https://cloud.redhat.com/openshift/install) a pre-built release and install the oc as well as kubectl binaries under /usr/bin. You will need to login with your Red Hat account and select the infrastructure provider where your OpenShift cluster is running. Remember to check the permissions and ensure the binaries have execute permission. You can also download command line tools for OpenShift from the OpenShift cluster itself under the question mark by your user in the UI.

### Install Operator Framework SDK
This training requires minimum v1.0.1 of the operator-sdk and ansible-operator binaries.

[Download](https://github.com/operator-framework/operator-sdk/releases) a pre-built release of both binaries and install them under /usr/bin.

## Exercise 1
In this exercise you will complete the following:
* Create Operator scaffolding in Ansible
* Add a simple debug print statement to the operator Ansible role
* Update the operator CR and add a parameter to toggle the print statement
* Test the operator using the ansible-runner

#### Create a new OpenShift project
```$ oc new-project operator-helloworld```

### Create Operator Scaffolding
Using the operator-sdk, create the scaffolding for your new operator. The operator-sdk will generate an Ansible role, create a new custom resource definition (CRD) and all the necessary k8s objects to install the operator. This is a two step process. First initialize and then create API.

```$ mkdir operator-helloworld```

```$ cd operator-helloworld```

```$ operator-sdk init --plugins=ansible --domain=hello.example.com```

```$ operator-sdk create api --group cache --version v1 --kind Hello --generate-role```

### Create Custom Resource Definition (CRD)
The operator-sdk will generate a CRD this will extend the k8s API and allow users to interact with the Operator through the API. Here we will install CRD in the current namespace operator-helloworld.

```$ make install```

#### Add Print Task to Operator Role
The operator framework implements Ansible roles. By default it will create a single role but you can certainly have many roles. Roles are mapped to the API endpoint of the CRD in the watches.yaml file. In this case we will be adding a print statement that will print some debug when a parameter toggle_message is set to true to the role.

```$ vi roles/hello/tasks/main.yml```

```
---
# tasks file for hello
- name: Hello World Task
  debug:
    msg: "Hello World! I live in a namespace called {{ ansible_operator_meta.namespace }}"
  when: toggle_message
```
#### Add parameter to the Operator Custom Resource
Here we will add the toggle_message parameter to the CR. Any parameters under the CR spec are automatically visible in Ansible. This is how you get input from your users. In addition as you may have noticed you can access CR metadata using the ansible_operator_meta parameter in ansible. In the above example that is the name os the namespace.

```$ vi config/samples/cache_v1_hello.yaml```

```
apiVersion: cache.hello.example.com/v1
kind: Hello
metadata:
  name: hello-sample
spec:
  toggle_message: true
```

#### Run Operator using ansible-runner
Now that we have implemented some tasks and our parameter we can run ther Operator locally using the ansible-runner to test it. 

```$ ansible-operator run local```

#### Create a hello customer resource
Open another terminal and create the CR. Once the CR is created, the Operator will execute the Ansible role and print our debug message.

```$ oc create -f config/samples/cache_v1_hello.yaml```

```
TASK [Hello World Task] ********************************
ok: [localhost] => {
    "msg": "Hello World! I live in a namespace called operator-helloworld"
}
```

## Exercise 2
In this exercise you will complete the following:
* Expand Ansible role to get the cluster domain name and save that in a fact
* Expand the Ansible role to deply a helloworld application
* Test the operator using the ansible-runner
* Deploy operator to namespace so it runs without the ansible-runner

### Update Ansible role to get cluster domain name and save as a fact
Here we will learn to use the k8s Ansible module to gather information we want to use later in our automation. In this case the cluster domain name. Append the following tasks to the Ansible role.

```$ vi roles/hello/tasks/main.yml```

```
- name: Get Application Domain from Cluster Ingress
  k8s_info:
    api_version: config.openshift.io/v1
    kind: Ingress
    name: cluster
  when: application_domain is undefined
  register: ingress

- name: Set Application Domain
  set_fact:
    application_domain: "{{ ingress.resources[0].spec.domain }}"
  when: application_domain is undefined

- name: Print application domain
  debug:
    msg: "Application domain is {{ application_domain }}"
```

#### Run Operator using ansible-runner
Each time Operator is started or something changes with our CRD the Ansible role will run and our changes will of course be executed.

```$ ansible-operator run local```

```
TASK [Print application domain] ********************************
ok: [localhost] => {
    "msg": "Application domain is apps.ocp4.keithtenzer.com"
}
```

### Update Ansible role to deploy hellowoworld application
Now we will learn to use the k8s Ansible module to deploy an application. We will deploy a helloworld application that prints to STDOUT. Notice the route is using the cluster domain we gathered in the previous step. In this step we will create a deployment, service and route for our helloworld application. Append the following tasks to the Ansible role.

```$ vi roles/hello/tasks/main.yml```

```
- name: Deploy helloworld service
  k8s:
    definition:
      apiVersion: v1
      kind: Service
      metadata:
        namespace: "{{ ansible_operator_meta.namespace }}"
        labels:
          app: helloworld
        name: helloworld
      spec:
        ports:
        - port: 8080
          targetPort: 8080
        selector:
          app: helloworld
          name: helloworld
      status:
        loadBalancer: {}

- name: Deploy helloworld app
  k8s:
    definition:
      kind: Deployment
      apiVersion: apps/v1
      metadata:
        name: helloworld
        namespace: "{{ ansible_operator_meta.namespace }}"
        labels:
          app: helloworld
      spec:
        replicas: 1
        strategy:
          type: RollingUpdate
        selector:
          matchLabels:
            app: helloworld
        template:
          metadata:
            labels:
              app: helloworld
              name: helloworld
          spec:
            containers:
            - image: openshift/hello-openshift
              imagePullPolicy: Always
              name: helloworld
              readinessProbe:
                httpGet:
                  path: /
                  port: 8080
                initialDelaySeconds: 60
                periodSeconds: 10
                timeoutSeconds: 60
              livenessProbe:
                httpGet:
                  path: /
                  port: 8080
                initialDelaySeconds: 120
                periodSeconds: 10
              ports:
              - containerPort: 8080
            restartPolicy: Always
        triggers:
        - type: ConfigChange

- name: Deploy helloworld route
  k8s:
    definition:
      apiVersion: route.openshift.io/v1
      kind: Route
      metadata:
        namespace: "{{ ansible_operator_meta.namespace }}"
        annotations:
          openshift.io/host.generated: "true"
        name: helloworld
      spec:
        host: "hello-{{ ansible_operator_meta.namespace }}.{{application_domain}}"
        to:
          kind: Service
          name: helloworld
          weight: 100
        port:
          targetPort: 8080
        wildcardPolicy: None
```

### Update role permissions
Since we are creating a service and route we need to add those permissions to the role.

Add services so we can create them.

```$ vi config/rbac/role.yaml```

```
  - apiGroups:
      - ""
    resources:
      - secrets
      - pods
      - pods/exec
      - pods/log
      - services
```

Append routes and ingress api groups so we can also manage those objects.
```
  - apiGroups:
    - ""
    - config.openshift.io
    resources:
    - ingresses
    verbs:
    - get
    - list
    - watch
  - apiGroups:
    - ""
    - route.openshift.io
    resources:
    - routes
    - routes/custom-host
    verbs:
    - create
    - delete
    - deletecollection
    - get
    - list
    - patch
    - update
    - watch
```
### Run Operator using ansible-runner
This time we should see the application being deployed. A single pod should start and a service/route should be created.

```$ ansible-operator run local```

### Test our deployment
To test simply use curl against the route URL. It does take a minute or so to start application.

```
$ oc get pods
NAME                         READY   STATUS    RESTARTS   AGE
helloworld-f9d964dcc-jgcmn   1/1     Running   0          70s
```

```
$ oc get routes
NAME         HOST/PORT                                             PATH   SERVICES     PORT   TERMINATION   WILDCARD
helloworld   hello-operator-helloworld.apps.ocp4.keithtenzer.com          helloworld   8080
```

```
$ curl http://hello-operator-helloworld.apps.ocp4.keithtenzer.com
Hello OpenShift!
```

## Exercise 3
In this exercise you will complete the following:
* Create a quay.io account
* Build image of our Operator and push it to quay.io
* Deploy Operator to OpenShift cluster

### Create Quay.io Account
Quay.io us a container registry provided by Red Hat. You can create your own account push container images to it. Each image can be public or private. To make images available to OpenShift you will need to make them public.

Go to [Quay.io](https://quay.io/) and create your own account if you don't have one.

### Build Operator image and push to quay.io

```$ sudo make docker-build docker-push IMG=quay.io/ktenzer/operator-helloworld:latest```

Make the operator-helloworld image in your quay.io account public. Log into quya.io, click on the image. Under settings (on the left) there is option to make the image public.

### Deploy Operator to OpenShift Cluster
By default the operator will be deployed to a project called operator-helloworld-system. You can change this by editing the ```config/default/kustomization.yaml``` file.

```$ make deploy IMG=quay.io/ktenzer/operator-helloworld:latest```

Check Operator Deployment

```$ oc get deployment -n operator-helloworld-system```

```
NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
helloworld-controller-manager   1/1     1            1           37s
```

### Deploy Helloworld Application using Operator
Using the Operator we just deployed into the operator-helloworld-system namespace we will now deploy the application using CR.

```$ oc create -f config/samples/cache_v1_hello.yaml -n operator-helloworld-system```

```$ oc get deployment -n operator-helloworld-system```

```
NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
helloworld                      1/1     1            1           12m
helloworld-controller-manager   1/1     1            1           12m
```

### Cleanup Application
Removing the CR will delete everything that was created by it since the objects are linked to the CR.

```$ oc delete hello hello-sample -n operator-helloworld-system```

### Cleanup Operator 
This will remove the Operator, CRD and all the roles.

```$ make undeploy```

Congrats if you got this far you are ready to write your own Operators in Ansible!
