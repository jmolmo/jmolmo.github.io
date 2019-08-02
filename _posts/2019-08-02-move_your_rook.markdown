---
layout: post
title:  "Move your Rook!"
date:   2019-08-02 16:53:13 +0200
categories: Rook K8 Ceph
---

![move_the_rook](/assets/images/darlene-cerna-iRL1P94z4Xc-unsplash.jpg)
Photo by Darlene Cerna on Unsplash

This is basically a step by step guide to show the process needed to have a kubernetes lab with several nodes working, the Rook operator installed and a Ceph test cluster created by Rook
Using this lab up i will explain how to do a very first modification in the Rook operator.


### 1. Create the k8 cluster

We need a flexible and quick way to create a k8's cluster for our lab. This means that we must create/delete/stop and modify the composition of the k8's cluster in a breeze.
@galexrt has done the hard work for us, and provide the right tool to setup our lab very quickly.
Basically it uses Vagrant to configure a set of vm machines where to install the  kubernetes master with the preferred number of nodes. We can do all the work using _make commands_ with different _targets_

So the first thing to do is to clone [k8s-vagrant-multi-node](https://github.com/galexrt/k8s-vagrant-multi-node) and to have a coffee while reading to understand how it works.

If you do not want to have a coffee, you can use these commands to have your k8's cluster (1 master and 3 nodes) up and running in just 5 minutes.

```
$ git clone git@github.com:galexrt/k8s-vagrant-multi-node.git
$ cd k8s-vagrant-multi-node
$ NODE_COUNT=3 make up -j4
```

NOTE:
The tool allows you to change several env vars, apart of making them _permanent_ modifying the _Makefile_
Ex. In my case, I wanted to use always CentOS images for the k8's nodes, so I introduce the following modification in the _Makefile_ file:

BOX_OS ?= centos


Check the creation of the different vm's and the k8's cluster health:

```
$ VBoxManage list runningvms
"k8s-vagrant-multi-node_node3_1564563685177_19234" {fcc23840-91df-43b8-ba72-2daf5ee296d8}
"k8s-vagrant-multi-node_master_1564563683178_24986" {6c86c268-0339-4331-a91f-117865815d27}
"k8s-vagrant-multi-node_node1_1564563683203_78317" {eaf1b72f-02e5-437b-af33-3cf8703f4708}
"k8s-vagrant-multi-node_node2_1564563683263_17543" {57231ef2-f216-4961-b9e5-3479385388e1}


$ kubectl get nodes
NAME      STATUS    ROLES     AGE       VERSION
master    Ready     master    14m       v1.15.1
node1     Ready     <none>    13m       v1.15.1
node2     Ready     <none>    13m       v1.15.1
node3     Ready     <none>    13m       v1.15.1

$ kubectl cluster-info
Kubernetes master is running at https://192.168.26.10:6443
KubeDNS is running at https://192.168.26.10:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

$ kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health":"true"}
```

### 2. Install Rook and create a Ceph cluster:

Rook is great!!... never was so easy to install a Ceph cluster.... k8's and operators provide the magic, Rook is the wizard!.

Once again, maybe you will enjoy a tea while reading the [quickstart guide](https://rook.github.io/docs/rook/master/ceph-quickstart.html).

If you do not like tea, you can use directly my commands:

First thing to do is to clone the repo in your Go src folder:

```
$ cd /home/jolmomar/go/src/github.com/
$ mkdir rook
$ cd rook
$ git clone git@github.com:rook/rook.git
```

Once we have the source code let's go to create the Ceph custom resources and the operator, using the _*.yaml_ files that you can find in the examples folder.

```
$ $ pwd
/home/jolmomar/go/src/github.com/rook/rook/cluster/examples/kubernetes/ceph

$ kubectl create -f common.yaml
$ kubectl create -f operator.yaml
```

Wait until operator and discovery pods(1 for node) will be running:

```
$ watch kubectl -n rook-ceph get pod
NAME                                  READY     STATUS    RESTARTS   AGE
rook-ceph-operator-56b7cdb77b-r7qlm   1/1       Running   0          5m27s
rook-discover-kftdr                   1/1       Running   0          3m33s
rook-discover-ndd7c                   1/1       Running   0          3m33s
rook-discover-xklgc                   1/1       Running   0          3m33s
```

Now, create the ceph cluster:

```
$ kubectl create -f cluster-test.yaml
```

Wait until all the pods will be running. In a few minutes you will have a basic functional Ceph cluster in K8's. This will be our lab!

```
$ kubectl -n rook-ceph get pod
NAME                                  READY     STATUS      RESTARTS   AGE
rook-ceph-agent-jv5z2                 1/1       Running     0          3m24s
rook-ceph-agent-qhj2x                 1/1       Running     0          3m24s
rook-ceph-agent-twm2b                 1/1       Running     0          3m24s
rook-ceph-mgr-a-7d9768cf76-swdsj      1/1       Running     0          2m36s
rook-ceph-mon-a-58bd7665fd-kz5f9      1/1       Running     0          2m54s
rook-ceph-operator-56b7cdb77b-r7qlm   1/1       Running     0          9m33s
rook-ceph-osd-0-685f69d9df-ktmpj      1/1       Running     0          2m
rook-ceph-osd-1-c98865cdf-f7lj9       1/1       Running     0          116s
rook-ceph-osd-2-b7c94b6bc-5fsxr       1/1       Running     0          114s
rook-ceph-osd-prepare-node1-hgvpx     0/2       Completed   0          2m7s
rook-ceph-osd-prepare-node2-ghxxm     0/2       Completed   0          2m7s
rook-ceph-osd-prepare-node3-ggjkf     0/2       Completed   1          2m7s
rook-discover-kftdr                   1/1       Running     0          7m39s
rook-discover-ndd7c                   1/1       Running     0          7m39s
rook-discover-xklgc                   1/1       Running     0          7m39s
```

### 3. Cleaning up things

Programming can be a dangerous activity sometimes. In the case that you make something crazy ... probably, you will like to clean up things and start again form scratch.... this is my "restart the world" recipe

For the Ceph Cluster managed by Rook you can follow this [instructions](https://rook.github.io/docs/rook/master/ceph-teardown.html), that I summarized here:

```
$ kubectl -n rook-ceph delete cephcluster rook-ceph

$ kubectl delete -f operator.yaml

$ kubectl delete -f common.yaml

$ kubectl -n rook-ceph get pod
```

And of course, you can even destroy the vm's with your k8 cluster if things went really, really wrong (in this case, please write a post explaining how did you get this awesome achievement!!)

Move to your _k8s-vagrant-multi-node_  folder and execute this command to clean the k8's master and the three nodes.

```
$ NODE_COUNT=3 make clean
```

### Move the Rook!! My first modification

The exciting moment arrived!. Let's modify the Rook operator and check our modification.

The Rook operator log will tell us about our modification, so let's check it, before any modification, just to take a look at it:

```
$ kubectl -n rook-ceph get pod
NAME                                  READY     STATUS              RESTARTS   AGE
rook-ceph-operator-6566dd4985-drrsw   1/1       Running             0          18s
rook-discover-5lnkz                   1/1       Running             0          6s
rook-discover-8mgvr                   0/1       ContainerCreating   0          6s
rook-discover-9j2v2                   0/1       ContainerCreating   0          6s
```

If we check the operator pod log we see that we have the following lines:

```
$ kubectl -n rook-ceph logs rook-ceph-operator-6566dd4985-drrsw


2019-07-31 11:41:54.722382 I | rookcmd: starting Rook v0.8.0-1294.g1b6fe840.dirty with arguments '/usr/local/bin/rook ceph operator'

2019-07-31 11:41:54.722508 I | rookcmd: flag values: ....
```

Lets try to insert a new log line when the operator starts:

We are going to do a very simple modification, just to add a new line in the Rook operator file when it starts. Modification will be done in the "rook/cmd/rook/rook/rook.go" file

```
// LogStartupInfo log the version number, arguments, and all final flag values (environment variable overrides have already been taken into account)
func LogStartupInfo(cmdFlags *pflag.FlagSet) {

	flagValues := flags.GetFlagsAndValues(cmdFlags, "secret|keyring")
	logger.Infof("starting Rook %s with arguments '%s'", version.Version, strings.Join(os.Args, " "))
	logger.Infof("flag values: %s", strings.Join(flagValues, ", "))
	logger.Infof("Hey!!!! -------------------- This is my very first line in Rook")
}
```

Lets build the new operator container. This can be done in the root folder of the project:

```
$ pwd
/home/jolmomar/go/src/github.com/rook/rook

$ make -j4 IMAGES='ceph' build
```

Check that we have our new container:

```
$ docker images
REPOSITORY                  TAG                 IMAGE ID            CREATED             SIZE
build-4bc9ed8c/ceph-amd64   latest              ccd3c6105d1d        37 seconds ago      921MB
```

We must use this new image to start the operator pod. This means that k8 nodes should be able to pull the image from a registry.
The easy option is to pull your modified image to one of your docker hub repos, and use it. So first thing to do is to create/use your docker hub account, and create a "rook" repo in docker hub.
Once we have the "infrastructure" we can push our new image and start to use it in our k8 nodes.

```
$ docker tag build-4bc9ed8c/ceph-amd64 jolmomar/rook:latest

$ docker push jolmomar/rook
The push refers to repository [docker.io/jolmomar/rook]
32011c0556a2: Pushed
8d8bce899a8e: Pushed
2f5e42981c8f: Pushed
c1e065038222: Layer already exists
d2774fc103ac: Layer already exists
d69483a6face: Layer already exists
latest: digest: sha256:b6053a2a5446944d9a0a3be50ec5b10f225ae2607ad013666bf08926ed77c563 size: 1579
```

To use this new image we must modify the operator.yaml file we use to manage the rook operator.

```
$ pwd
/home/jolmomar/go/src/github.com/rook/rook/cluster/examples/kubernetes/ceph
```

Modify the file _operator.yaml_, to include the _path_ to your Rook operator image:

```
...
spec:
  serviceAccountName: rook-ceph-system
  containers:
  - name: rook-ceph-operator
    image: jolmomar/rook
...
```

Once modified the "operator.yaml" file, lets go to try it. First thing to do is to delete the previous operator pod:

```
$ kubectl -n rook-ceph get pod
NAME                                  READY     STATUS    RESTARTS   AGE
rook-ceph-operator-6566dd4985-drrsw   1/1       Running   0          154m

$ kubectl delete -f operator.yaml
deployment.apps "rook-ceph-operator" deleted
```

Now let's go to create again the operator pod, this time the image for it is the modified one, and it is available in our "rook" docker hub repo.

```
$ kubectl create -f operator.yaml
deployment.apps/rook-ceph-operator created

$ kubectl -n rook-ceph get pod
NAME                                  READY     STATUS    RESTARTS   AGE
rook-ceph-operator-6566dd4985-z4s9r   1/1       Running   0          21s
```

And the last step is just to check that our log line appears in the log file

```
$ kubectl -n rook-ceph logs rook-ceph-operator-6566dd4985-z4s9r
2019-07-31 14:18:52.874209 I | rookcmd: starting Rook v0.8.0-1294.g1b6fe840.dirty with arguments '/usr/local/bin/rook ceph operator'
2019-07-31 14:18:52.874385 I | rookcmd: flag values: ...
...
2019-07-31 14:18:52.874395 I | rookcmd: Hey!!!! -------------------- This is my very first line in Rook
2019-07-31 14:18:52.874401 I | cephcmd: starting operator
2019-07-31 14:18:52.907075 I | op-discover: rook-discover daemonset already exists, updating ...
```

And this is all... now you have the power to modify and improve the Rook operator.

Is time to contribute your awesome ideas to the Rook project, the difficult part, to have the idea is done, so the only thing you need to do is to follow [the guide to contribute](https://github.com/rook/rook/blob/master/CONTRIBUTING.md)
