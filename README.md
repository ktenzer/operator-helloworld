# Operator Framework using Ansible Training

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

### Install Operator Framework SDK

You can simply [Downloads](https://github.com/operator-framework/operator-sdk/releases) a pre-built release and install it under /usr/local/bin.

## Exercise 1
In this exercise you will complete the following:
* Create Operator scaffolding in Ansible
* Add a simple debug print statement to the operator Ansible role
* Update the operator CR and add a parameter to toggle the print statement
* Test the operator using the ansible-runner

#### Create a new OpenShift project
```$ oc new-project operator-helloworld```

### Create Operator Scaffolding
Using the operator-sdk, create the scaffolding for your new operator. The operator-sdk will generate an Ansible role, create a new custom resource definition (CRD) and all the necessary k8s objects to install the operator.
``` operator-sdk new operator-helloworld --api-version=hello.example.com/v1alpha1 --kind=Hello --type=ansible```

### Create Custom Resource Definition (CRD)
The operator-sdk will generate a CRD this will extend the k8s API and allow users to interact with the Operator through the API.

```$ cd operator-helloworld```

```$ oc create -f deploy/crds/hello.example.com_hellos_crd.yaml```

### Create role, role binding and service
The operator-sdk will autogenerate these k8s objects so we just need to create them in our project.

```$ oc create -f deploy/role.yaml```

```$ oc create -f deploy/role_binding.yaml```

```$ oc create -f deploy/service_account.yaml```

#### Add Print Task to Operator Role
The operator framework implements Ansible roles. By default it will create a single role. Roles are mapped to the API endpoint of the CRD in the watches.yaml file. You can have many roles as well. If you have more roles you will typically use the default role to import and execute other roles that handle specific tasks. In this case we will be adding a print statement that will print some debug when a parameter toggle_message is set to true.

```$ vi roles/hello/tasks/main.yml```

```---
# tasks file for hello
- name: Hello World Task
  debug:
    msg: "Hello World! I live in a namespace called {{ meta.namespace }}"
  when: toggle_message

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
#### Add parameter to the Operator Custom Resource
By default the auto-generated CR is not parameterized. Here we will add the toggle_message parameter. As you can see above any parameters under the spec are automatically visible in Ansible. This is how you get input from your users.

```$ vi deploy/crds/hello.example.com_v1alpha1_hello_cr.yaml```

```apiVersion: hello.example.com/v1alpha1
kind: Hello
metadata:
  name: example-hello
spec:
  # Add fields here
  toggle_message: true
```

#### Run Operator using ansible-runner
Now that we have implemented some tasks and our parameter we can run ther Operator locally using the ansible-runner to test it. 

```$ operator-sdk run --local```

#### Create a hello customer resource
Open another terminal and create the CR. You should then see the CR created, the Operator will notice that and print our debug message.

```$ oc create -f deploy/crds/hello.example.com_v1alpha1_hello_cr.yaml```

```TASK [Hello World Task] ********************************
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
Here we will learn to use the k8s Ansible module to gather information we want to use later in our automation. In this case the cluster domain name.

```$ vi roles/hello/tasks/main.yml```

```- name: Get Application Domain from Cluster Ingress
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

```$ operator-sdk run --local```

### Update Ansible role to deploy hellowoworld application
Now we will learn to use the k8s Ansible module to deploy an application. Notice the route is using the cluster domain we gathered in the previous step. In this step we will create a deployment, service and route for our helloworld application.

```$ vi roles/hello/tasks/main.yml```

```- name: Deploy helloworld service
  k8s:
    definition:
      apiVersion: v1
      kind: Service
      metadata:
        namespace: "{{ meta.namespace }}"
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
        namespace: "{{ meta.namespace }}"
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
        namespace: "{{ meta.namespace }}"
        annotations:
          openshift.io/host.generated: "true"
        name: helloworld
      spec:
        host: "hello-{{ meta.namespace }}.{{application_domain}}"
        to:
          kind: Service
          name: helloworld
          weight: 100
        port:
          targetPort: 8080
        wildcardPolicy: None
```

### Run Operator using ansible-runner
This time we should see the application being deployed. A single pod should start and a service/route should be created.

```$ operator-sdk run --local```

### Test our deployment
To test simply use curl against the route URL.

```$ curl http://hello-operator-helloworld.apps.ocp4.keithtenzer.com
Hello OpenShift!
```

### Deply Operator into namespace
The operator-sdk creates a deployment for deplying the operator. We simply need to create it.

```$ oc create -f deploy/operator.yaml```

Congrats if you got this far you are ready to write your own Operators!
