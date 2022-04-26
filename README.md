# mirror-operator

Reference:
[OpenShift Pages](https://docs.openshift.com/container-platform/4.10/operators/admin/olm-restricted-networks.html)

## install grpcurl (third-party command-line tool)

```
[lab-user@bastion mirror]$ wget https://github.com/fullstorydev/grpcurl/releases/download/v1.8.6/grpcurl_1.8.6_linux_x86_64.tar.gz

[lab-user@bastion mirror]$ tar xvf grpcurl_1.8.6_linux_x86_64.tar.gz 

[lab-user@bastion mirror]$ sudo mv grpcurl /usr/local/sbin/.

[lab-user@bastion mirror]$ grpcurl --version
grpcurl v1.8.6

[lab-user@bastion mirror]$ wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/latest-4.10/opm-linux.tar.gz

[lab-user@bastion mirror]$ tar xvf opm-linux.tar.gz
opm

[lab-user@bastion mirror]$ sudo mv opm /usr/local/sbin/.

[lab-user@bastion mirror]$ opm version
Version: version.Version{OpmVersion:"b576eef43", GitCommit:"b576eef43e0d4c3745435a068b8bf42ec347eda3", BuildDate:"2022-04-11T10:00:52Z", GoOs:"linux", GoArch:"amd64"}

```

**Podman need to be >1.8 for mirroring**

Manually downlaod and upload podman 1.9.3 to the server.

```
[root@bastion ~]# podman version
Version:            1.6.4

[root@bastion ~]# yum localinstall -y /tmp/podman-1.9.3-2.module+el8.2.1+6867+366c07d6.x86_64.rpm

[root@bastion ~]# podman version
Version:            1.9.3

```


## Create local image repo ##

```
[root@bastion ~]# mkdir -p /opt/registry/{auth,certs,data}

[root@bastion ~]# cd /opt/registry/certs/

[root@bastion certs]# openssl genrsa -out Ext-Registry-CA.key 2048

[root@bastion certs]# openssl req -x509 -new -nodes -key Ext-Registry-CA.key -sha256 -days 1095 -out Ext-Registry-CA.pem
-----
Country Name (2 letter code) [XX]:SG
State or Province Name (full name) []:SG
Locality Name (eg, city) [Default City]:SG
Organization Name (eg, company) [Default Company Ltd]:Red Hat
Organizational Unit Name (eg, section) []:GPS
Common Name (eg, your name or your server's hostname) []:CA
Email Address []:
[root@bastion certs]# 

[root@bastion certs]# openssl req -new -key Ext-Registry-CA.key -out bastion.csr
-----
Country Name (2 letter code) [XX]:SG
State or Province Name (full name) []:SG
Locality Name (eg, city) [Default City]:SG
Organization Name (eg, company) [Default Company Ltd]:Red Hat
Organizational Unit Name (eg, section) []:GPS
Common Name (eg, your name or your server's hostname) []:bastion.n2p5z.internal
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
[root@bastion certs]# 

[root@bastion certs]# vi san.txt
subjectAltName = DNS:bastion.n2p5z.internal, DNS:localhost

[root@bastion certs]# openssl x509 -req -in bastion.csr -CA Ext-Registry-CA.pem -CAkey Ext-Registry-CA.key -CAcreateserial -out bastion.pem -days 1024 -sha256 -extfile san.txt

[root@bastion certs]# cp /opt/registry/certs/Ext-Registry-CA.pem /etc/pki/ca-trust/source/anchors/

[root@bastion certs]# update-ca-trust

[root@bastion certs]# htpasswd -bBc /opt/registry/auth/htpasswd admin redhat123

[root@bastion certs]# podman run --name mirror-registry -p 5000:5000 \
   	-v /opt/registry/data:/var/lib/registry:z \
   	-v /opt/registry/auth:/auth:z \
   	-e "REGISTRY_AUTH=htpasswd" \
   	-e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
   	-e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
   	-v /opt/registry/certs:/certs:z \
   	-e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/bastion.pem \
   	-e REGISTRY_HTTP_TLS_KEY=/certs/Ext-Registry-CA.key \
   	-e REGISTRY_COMPATIBILITY_SCHEMA1_ENABLED=true \
  	-d docker.io/library/registry:2.7.1

[root@bastion certs]# podman ps
CONTAINER ID  IMAGE                             COMMAND               CREATED             STATUS             PORTS                   NAMES
0cb00cdd9abf  docker.io/library/registry:2.7.1  /etc/docker/regis...  About a minute ago  Up 59 seconds ago  0.0.0.0:5000->5000/tcp  mirror-registry

[root@bastion certs]# curl -u admin:redhat123 https://localhost:5000/v2/_catalog
{"repositories":[]}

[root@bastion certs]# podman login localhost:5000
Username: admin
Password: redhat123
Login Succeeded!


```

## Mirror GPU Operator ##

>**GPU Operator is in certified-operator-index instead of redhat-operator-index**

https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/openshift/mirror-gpu-ocp-disconnected.html

The four primary official indexes the OpenShift Container Platform uses are:

- registry.redhat.io/redhat/certified-operator-index:v4.9
- registry.redhat.io/redhat/redhat-operator-index:v4.9
- registry.redhat.io/redhat/community-operator-index:v4.9
- registry.redhat.io/redhat/redhat-marketplace-index:v4.9


**Authenticate with registry.redhat.io**

```
[root@bastion certs]# podman login registry.redhat.io --tls-verify=false

```

**Run the source index image that you want to prune in a container**

Run on one session

```
[root@bastion certs]# oc get clusterversion
NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.9.0     True        False         13h     Cluster version is 4.9.0

[root@bastion certs]# podman run -p50051:50051 -it registry.redhat.io/redhat/certified-operator-index:v4.9

```

## Mirroring an Operator catalog

**Pruning & Push an index image**

Open new session and run

```
[root@bastion mirror-operator]# grpcurl -plaintext localhost:50051 api.Registry/ListPackages > packages.out

[root@bastion mirror-operator]# grep -1 -i gpu packages.out
{
  "name": "gpu-operator-certified"
}

# Geneate the pruned the source index image
[root@bastion mirror-operator]# opm index prune -f registry.redhat.io/redhat/certified-operator-index:v4.9 -p gpu-operator-certified -t localhost:5000/olm-mirror/certified-operator-index:v4.9

# push the new index image to the target registry
[root@bastion mirror-operator]# podman push localhost:5000/olm-mirror/certified-operator-index:v4.9

[root@bastion mirror-operator]# curl -u admin:redhat123 https://localhost:5000/v2/_catalog
{"repositories":["olm-mirror/certified-operator-index"]}

[root@bastion mirror-operator]# curl -u admin:redhat123 https://localhost:5000/v2/olm-mirror/certified-operator-index/tags/list
{"name":"olm-mirror/certified-operator-index","tags":["v4.9"]}

```
