# Operator Framework using Ansible Training

## Setup Development Environment
There are some prerequisites needed to develop and build an operator using Ansible. Also this guide and the operator-sdk assume you know Ansible roles. If you aren not yet up to speed please read about Ansible roles before proceeding.

## Install Docker 17.03+

Add the docker ce repositories. Note: you can also use podman and buildah instead of Docker for those that want a complete and clean divorce.

```$ sudo dnf -y install dnf-plugins-core```

```$ sudo dnf config-manager \
    --add-repo \
    https://download.docker.com/linux/fedora/docker-ce.repo```

## Install docker-ce

```$ sudo dnf -y install docker-ce docker-ce-cli```
```$ sudo systemctl start docker```
```$ sudo systemctl enable docker```

## Install Ansible and Module Dependencies

### Install ansible

```$ dnf install ansible```

The Ansible runner and http runner is used to run a local version of the operator. This is very useful for development and testing.

```$ pip3 install --user ansible-runner```
```$ pip3 install --user ansible-runner-http```

### Install required python modules

```$ pip3 install --user requests $ pip3 install --user openshift```

## Install Operator Framework SDK

You can simply [Downloads](https://github.com/operator-framework/operator-sdk/releases) a pre-built release and install it under /usr/local/bin.
