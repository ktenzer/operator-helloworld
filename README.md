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

```$ dnf install ansible```

The Ansible runner and http runner is used to run a local version of the operator. This is very useful for development and testing.

```$ pip3 install --user ansible-runner```
```$ pip3 install --user ansible-runner-http```

#### Install required python modules

```$ pip3 install --user requests $ pip3 install --user openshift```

### Install Operator Framework SDK

You can simply [Downloads](https://github.com/operator-framework/operator-sdk/releases) a pre-built release and install it under /usr/local/bin.

## Exercise 1
In this exercise you will complete the following:
* Create Operator scaffolding in Ansible
* Add a simple debug print statement to the operator Ansible role
* Update the operator CR and add a parameter to toggle the print statement
* Test the operator using the ansible-runner

### Create Operator Scaffolding

#### Clone GitHub repository
```$ git clone https://github.com/ktenzer/operator-helloworld.git```

```$ cd operator-helloworld```

#### Add Print Task to Operator Role
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
```$ vi deploy/crds/hello.example.com_v1alpha1_hello_cr.yaml```

```apiVersion: hello.example.com/v1alpha1
kind: Hello
metadata:
  name: example-hello
spec:
  # Add fields here
  toggle_message: true
```

#### Create a new OpenShift project
```$ oc new-project operator-helloworld```

#### Run Operator using ansible-runner
```$ operator-sdk run --local```

#### Create a hello customer resource
```$ oc create -f deploy/crds/hello.example.com_v1alpha1_hello_cr.yaml```
