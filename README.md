# mirror-operator

Reference:
[OpenShift Pages](https://docs.openshift.com/container-platform/4.10/operators/admin/olm-restricted-networks.html)

## install grpcurl (third-party command-line tool)

```
[lab-user@bastion mirror]$ wget https://github.com/fullstorydev/grpcurl/releases/download/v1.8.6/grpcurl_1.8.6_linux_x86_64.tar.gz

[lab-user@bastion mirror]$ tar xvf grpcurl_1.8.6_linux_x86_64.tar.gz 

[lab-user@bastion mirror]$ sudo mv grpcurl /usr/local/bin/.

[lab-user@bastion mirror]$ grpcurl --version
grpcurl v1.8.6

[lab-user@bastion mirror]$ wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/latest-4.10/opm-linux.tar.gz

[lab-user@bastion mirror]$ tar xvf opm-linux.tar.gz
opm

[lab-user@bastion mirror]$ sudo mv opm /usr/local/bin/.

[lab-user@bastion mirror]$ opm version
Version: version.Version{OpmVersion:"b576eef43", GitCommit:"b576eef43e0d4c3745435a068b8bf42ec347eda3", BuildDate:"2022-04-11T10:00:52Z", GoOs:"linux", GoArch:"amd64"}

```

**Authenticate with registry.redhat.io**

```
[lab-user@bastion mirror]$ podman login -u YYYY -p XXXXX registry.redhat.io

```

**Run the source index image that you want to prune in a container**

Run on one session

```
[lab-user@bastion mirror]$ podman run -p50051:50051 -it registry.redhat.io/redhat/redhat-operator-index:v4.10


```

**Cannot find gpu operator**
Open new session and run
```
[lab-user@bastion mirror]$ grpcurl -plaintext localhost:50051 api.Registry/ListPackages > packages.out

[lab-user@bastion mirror]$ grep -i gpu packages.out 

```

https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/openshift/mirror-gpu-ocp-disconnected.html

The four primary official indexes the OpenShift Container Platform uses are:

- registry.redhat.io/redhat/certified-operator-index:v4.9
- registry.redhat.io/redhat/redhat-operator-index:v4.9
- registry.redhat.io/redhat/community-operator-index:v4.9
- registry.redhat.io/redhat/redhat-marketplace-index:v4.9

```

[lab-user@bastion mirror]$ podman run -p50051:50051 -it registry.redhat.io/redhat/certified-operator-index:v4.10

```

```
[lab-user@bastion mirror]$ grpcurl -plaintext localhost:50051 api.Registry/ListPackages > packages.out

[lab-user@bastion mirror]$ vim packages.out
...
{
  "name": "gpu-operator-certified"
}
...


```
