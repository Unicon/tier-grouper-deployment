# tier-grouper-deployment

# Env Prep

## Clean Centos 7 install

- user account that can sudo
- SSH into the server for copy/paste, etc.

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

sudo yum install docker-ce 

#https://github.com/moby/moby/issues/16137#issuecomment-271615192
sudo systemctl stop firewalld
sudo systemctl disable firewalld

sudo systemctl enable docker
sudo systemctl start docker
```

## Install Git and pull project source

```
sudo yum install git

git clone https://github.com/Unicon/tier-grouper-deployment.git
cd tier-grouper-deployment
```


# Installing Swarm

```
sudo docker swarm init
```

# Start Registry

```
sudo docker service create --name registry -p 5000:5000 registry:2
sudo docker service ls
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
```

## Populate Database

```
sudo docker run -it --rm \
  --mount type=bind,src=$(pwd)/grouper.hibernate.properties,dst=/run/secrets/grouper_grouper.hibernate.properties \
  --network internal \
  tier/grouper gsh -registry -check -runscript -noprompt

cd ..
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


# Build Base Image

```
cd base

sudo docker build --tag=localhost:5000/organization/grouper-base .
sudo docker push localhost:5000/organization/grouper-base

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
  --secret source=grouper.hibernate.properties,target=grouper_grouper.hibernate.properties \
  --secret source=subject.properties,target=grouper_subject.properties \  
  --secret host-key.pem \
  --config source=shibboleth2.xml,target=/etc/shibboleth/shibboleth2.xml \
  --config source=host-cert.pem,target=/etc/pki/tls/certs/host-cert.pem \
  --config source=host-cert.pem,target=/etc/pki/tls/certs/cachain.pem \
  localhost:5000/organization/grouper-ui
  
sudo docker service list
```

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
  --secret source=grouper.hibernate.properties,target=grouper_grouper.hibernate.properties \
  --secret source=subject.properties,target=grouper_subject.properties \
  --secret host-key.pem \
  --config source=host-cert.pem,target=/etc/pki/tls/certs/host-cert.pem \
  --config source=host-cert.pem,target=/etc/pki/tls/certs/cachain.pem \
  localhost:5000/organization/grouper-ws

sudo docker service list
```

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
sudo docker service rm registry 
sudo docker network rm internal
```