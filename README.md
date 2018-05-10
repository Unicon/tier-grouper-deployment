# Deploying the TIER Grouper Image using Docker Swarm

This is the companion repository for the 4-part video tutorial that demonstrates how to deploy the TIER Grouper Docker image. In the video series, John Gasper, IAM Consultant, demonstrates:

1. Create a simple Docker Swarm.
1. Create an organizational base image.
1. Populate the Grouper database.
1. Create custom images for each component (vs a single common image).
1. Create Docker Secrets and Docker Configs.
1. Starting Swarm Services: Grouper Daemon, UI, and WS.
1. Discussing next steps for moving forward in your deployment.

The videos are:

1. Session 1: Grouper Environment Prep: https://youtu.be/750J5UBTctw
1. Session 2: Grouper Database and Configs: https://youtu.be/agX-cm4-Okg
1. Session 3: Grouper Services: https://youtu.be/HQ0qjOysexA
1. Session 4: Grouper Continuing Forward: https://youtu.be/eMAL_RamYPc

> This tutorial was funded through Unicon's Grouper Open Source Support program. We thank our clients that are members of the program for their support that made this project possible. For more information about the Grouper OSS program, please see https://unicon.net/opensource/grouper.

# Env Prep

## Clean Centos 7 install

- 4 processors 
- 4-6 gb RAM
- 64 gb harddrive (we only use about 10gb)
- all host/guest shared components turned off
- user account that can sudo

## On the new machine

```
ip addr show
```

- SSH into the server (copy/paste, etc. is easier)
- setup grouper.example.edu and idp.example.edu hostnames

## VM Tools installed 

```
sudo mount -t iso9660 /dev/sr0 /mnt
cd /mnt
sudo ./install
sudo reboot
```

## Fully updated

```
sudo yum update
```

## Docker installed

```
sudo yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2

sudo yum-config-manager \
    --add-repo https://download.docker.com/linux/centos/docker-ce.repo

sudo yum install -y docker-ce 

#https://github.com/moby/moby/issues/16137#issuecomment-271615192
sudo systemctl stop firewalld
sudo systemctl disable firewalld

sudo systemctl enable docker
sudo systemctl start docker
```

## Install Git and pull project source

```
sudo yum install -y git

git clone https://github.com/Unicon/tier-grouper-deployment.git
cd tier-grouper-deployment
```


# Installing Swarm

```
sudo docker swarm init
```

# Start Registry

```
sudo docker container run -d --name registry -p 5000:5000 registry:2
sudo docker container ps
```

```
curl http://localhost:5000/v2/_catalog
```

# Ancillary Services

## Start Database, LDAP, and Shibboleth IdP Services

```
cd ancillary

sudo docker network create --driver overlay --scope swarm --attachable internal
sudo docker stack deploy anc -c stack.yml
sudo docker service ls
cd ..
```

# Build Base Image

```
cd base

sudo docker build --tag=localhost:5000/organization/grouper-base .
sudo docker push localhost:5000/organization/grouper-base

cd ..
```

## Populate Database

```
sudo docker container run -it --rm \
  --mount type=bind,src=$(pwd)/configs-and-secrets/grouper.hibernate.properties,dst=/run/secrets/grouper_grouper.hibernate.properties \
  --mount type=bind,src=$(pwd)/configs-and-secrets/subject.properties,dst=/run/secrets/grouper_subject.properties \
  --network internal \
  localhost:5000/organization/grouper-base gsh -registry -check -runscript -noprompt

sudo docker container run -it --rm \
  --mount type=bind,src=$(pwd)/configs-and-secrets/grouper.hibernate.properties,dst=/run/secrets/grouper_grouper.hibernate.properties \
  --mount type=bind,src=$(pwd)/configs-and-secrets/subject.properties,dst=/run/secrets/grouper_subject.properties \
  --network internal \
  localhost:5000/organization/grouper-base gsh
```

```
grouperSession = GrouperSession.startRootSession();
addMember("etc:sysadmingroup","jgasper");
:quit
```

## Configs and Secrets

```
cd configs-and-secrets

sudo docker secret create grouper.hibernate.properties grouper.hibernate.properties
sudo docker secret create subject.properties subject.properties
sudo docker secret create host-key.pem host-key.pem
sudo docker config create shibboleth2.xml shibboleth2.xml
sudo docker config create host-cert.pem host-cert.pem

cd ..
```





# Create services

## Daemon

```
cd daemon

sudo docker build --tag=localhost:5000/organization/grouper-daemon .
sudo docker push localhost:5000/organization/grouper-daemon

cd ..
```

```
sudo docker service create --detach --name=daemon \
  --network internal \
  --secret source=grouper.hibernate.properties,target=grouper_grouper.hibernate.properties \
  --secret source=subject.properties,target=grouper_subject.properties \
  localhost:5000/organization/grouper-daemon

sudo docker service list
```

## UI

```
cd ui

sudo docker build --tag=localhost:5000/organization/grouper-ui .
sudo docker push localhost:5000/organization/grouper-ui

cd ..
```

```
sudo docker service create --detach --name=ui \
  --network internal \
  --publish 443:443 \
  --secret source=grouper.hibernate.properties,target=grouper_grouper.hibernate.properties \
  --secret source=subject.properties,target=grouper_subject.properties \
  --secret source=host-key.pem,target=host-key.pem \
  --config source=shibboleth2.xml,target=/etc/shibboleth/shibboleth2.xml \
  --config source=host-cert.pem,target=/etc/pki/tls/certs/host-cert.pem \
  --config source=host-cert.pem,target=/etc/pki/tls/certs/cachain.pem \
  localhost:5000/organization/grouper-ui
  
sudo docker service list
```

<https://<hostname_or_ip>/grouper>

## WS

```
cd ws

sudo docker build --tag=localhost:5000/organization/grouper-ws .
sudo docker push localhost:5000/organization/grouper-ws

cd ..
```

```
sudo docker service create --detach --name=ws \
  --network internal \
  --publish 8443:443 \
  --secret source=grouper.hibernate.properties,target=grouper_grouper.hibernate.properties \
  --secret source=subject.properties,target=grouper_subject.properties \
  --secret host-key.pem \
  --config source=host-cert.pem,target=/etc/pki/tls/certs/host-cert.pem \
  --config source=host-cert.pem,target=/etc/pki/tls/certs/cachain.pem \
  localhost:5000/organization/grouper-ws

sudo docker service list
```

<https://<hostname_or_ip>:8443/grouper-ws/status?diagnosticType=db>

# Resetting the Env

## Grouper Stuff

```
sudo docker service rm daemon
sudo docker service rm ui
sudo docker service rm ws

sudo docker secret rm grouper.hibernate.properties
sudo docker secret rm subject.properties
sudo docker secret rm host-key.pem
sudo docker config rm shibboleth2.xml
sudo docker config rm host-cert.pem
```

## Everything else

```
sudo docker stack rm anc
sudo docker container rm -f registry 
sudo docker network rm internal
```

# Bonus: Using a stack file to spin up the env

Do you understand the `docker service` and `docker secret` subcommands? Try using the short-cut `stack.yml` file to save you some time.

> You still need to have the internal network defined, the DB populated, and an image registry started... along with a directory and idp (using the ancillary ones or external ones).

Start up all the Grouper components and verify:

```
sudo docker stack deploy grouper -c stack.yml
sudo docker stack ls
sudo docker service ls
sudo docker secret ls
```

Shutdown the Grouper componenets and verify:

```
sudo docker stack rm grouper
sudo docker stack ls
sudo docker service ls
sudo docker secret ls
```


# Bonus: Adding a new secret (or config) to an existing service

```
sudo docker service update daemon \
--secret-add source=grouper-loader.properties,target=grouper_grouper-loader.properties
sudo docker service update ui \
--secret-add source=grouper-loader.properties,target=grouper_grouper-loader.properties
sudo docker service update ws \
--secret-add source=grouper-loader.properties,target=grouper_grouper-loader.properties
```

# Bonus: Updating an Existing Secret

Let's assuming we need to change the `grouper-loader.properties` secret. First we update the grouper-loader.properties file on the host with the desired changes. 

> Note: `-2` is an arbitrary extension. If additional changes were being made the `--secret-rm` parameter would also need to be updated to include that last arbitrary extension. But the `target=grouper_grouper-loader.properties` remains the same.

1. Add the new secret file:
    ```
    sudo docker secret create grouper-loader-2.properties grouper-loader.properties
    ```

1. Update the service removing the old secret and adding the new secret:
    ```
    sudo docker service update daemon \
    --secret-add source=grouper-loader-2.properties,target=grouper_grouper-loader.properties \
    --secret-rm grouper-loader.properties
    sudo docker service update ui \
    --secret-add source=grouper-loader-2.properties,target=grouper_grouper-loader.properties \
    --secret-rm grouper-loader.properties
    sudo docker service update ws \
    --secret-add source=grouper-loader-2.properties,target=grouper_grouper-loader.properties \
    --secret-rm grouper-loader.properties
    ```

1. At some point remove the old secret (this will prevent a `rollback` operation from working):
    ```
    sudo docker secret rm grouper-loader.properties
    ```

# Bonus: Rolling Back a Service Configuration Change

```
sudo docker service rollback daemon
sudo docker service rollback ui
sudo docker service rollback ws
```

# Bonus: Kubernetes Deployment Configuration

1. Create the multi-part secret and configmaps:
    ```
    cd configs-and-secrets
    kubectl create secret generic grouper --from-file=grouper.hibernate.properties --from-file=subject.properties --from-file=host-key.pem
    kubectl create configmap shibboleth2.xml --from-file=shibboleth2.xml
    kubectl create configmap host-certs --from-file=host-cert.pem --from-file=cachain.pem=host-cert.pem

    cd..
    ```
    
2. Apply the config    
    ```
    kubectl apply -f kubernetes.yaml
    ```
       
