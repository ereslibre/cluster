# kubernetes-cluster-vagrant

* [What](#what)
* [Requirements](#requirements)
* [Building Kubernetes artifacts](#building-kubernetes-artifacts)
* [Base box](#base-box)
* [Building a cluster](#building-a-cluster)
* [Deploying changes to the cluster while it's running](#deploying-changes-to-the-cluster-while-its-running)
* [HA deployments (multi master)](#ha-deployments-multi-master)
* [Destroying the cluster](#destroying-the-cluster)
* [License](#license)

## What

This project intends to make it very easy to push your changes to the Kubernetes
source code into a cluster.

It uses [Vagrant](https://www.vagrantup.com/) to create the different clusters,
and allows you to have different cluster [profiles](profiles).

The idea behind this project is to create a base box that downloads all the
required dependencies. This way, it's possible to completely remove network
latency (or work completely offline if required) from your workflow.

The idea is to also automate the whole process completely, allowing for the
automation to be disabled in case you as a Kubernetes developer want to perform
some steps manually.

You can push your changes on the Kubernetes source code to the cluster when you
are creating the cluster, or when it's already running.

Both single master and multi master deployments are supported. In the multi
master deployment case, a load balancer will be created in a separate machine.

It's also possible to run several clusters at the same time, as long as they are
different profiles and their cluster and/or machine names don't clash, as well
as the IP addresses assigned to them.

## Requirements

* [Virtualbox](https://www.virtualbox.org/)
* [Vagrant](https://www.vagrantup.com/)
* Kubernetes cloned under `$GOPATH/src/k8s.io/kubernetes`

## Building Kubernetes artifacts

This project can help you to build Kubernetes artifacts with a single command,
it's handy if you don't want to leave this directory. The tasks created by this
project will launch bazel in a container in your host. This container will be
kept running so we leave the bazel instance running inside the container.

If you need to modify autogenerated files bazel won't do this for you, so in
this case you still need to go to your `$GOPATH/src/k8s.io/kubernetes` and run
a regular `make` command to regenerate the autogenerated files.

Within this project directory you can run several targets in order for artifacts
to be recreated:

* `make debs`
  * Will regenerate the debs based targets:
    * `cri-tools`
    * `kubeadm`
    * `kubectl`
    * `kubelet`
    * `kubernetes-cni`

* `make images`
  * Will regenerate the container images targets:
    * `kube-apiserver`
    * `kube-controller-manager`
    * `kube-proxy`
    * `kube-scheduler`

* `make artifacts`
  * Will run:
    * `make debs`
    * `make images`

After you have generated the artifacts you can read how to create a cluster
using these artifacts or how to deploy them on an already running cluster:

* [Building a cluster](#building-a-cluster)
* [Deploying changes to the cluster while it's running](#deploying-changes-to-the-cluster-while-its-running)

## Base box

The base box will contain some general images that will be used later when
creating the cluster. These images are defined in
[`container_images.json`](base-box/configs/container_images.json). At the
moment they are:

* `coredns`
* `etcd`
* `flannel`
* `pause`

Also, the following images inside
`$GOPATH/src/k8s.io/kubernetes/_output/release-images/amd64` are also
copied and loaded in the base box:

* `kube-apiserver`
* `kube-controller-manager`
* `kube-proxy`
* `kube-scheduler`

Aside from that, the following packages inside
`$GOPATH/src/k8s.io/kubernetes/bazel-bin/build/debs` are copied and
installed in the base box too:

* `cri-tools`
* `kubeadm`
* `kubectl`
* `kubelet`
* `kubernetes-cni`

Building the base box requires internet connection for installing some packages
and the first set of containers (the ones defined in
[`container_images.json`](base-box/configs/container_images.json)).

Once the base box is built, it is saved as a Vagrant box named
`kubernetes-vagrant`.

The idea behind the base box is that it contains everything needed to bring up
a cluster in an offline environment, so if you are temporarily working on slow
networks it won't affect your workflow.

When you create a cluster, you can override whatever you want (e.g. `kubeadm`
and `kubelet` packages), that will be installed on top of the ones included by
the base box. This allows you to create clusters faster, only installing the
packages you modified and checking the results.

From time to time you'll need to rebuild the base box, but it's an action that
you won't perform nearly as often as creating machines for testing your changes
on the Kubernetes source code.

## Building a cluster

Building a cluster is trivial, all you need is to point to a specific profile.
Take into account that this process is completely offline and doesn't need to
download any information from the internet.

You can just run `make` with the `PROFILE` envvar set to a profile name that
exists inside the [profiles](profiles) folder, or to a full path containing
the profile you want.

```
~/p/kubernetes-cluster-vagrant (master) > time PROFILE=bootstrap/1-master-1-worker make
make -C base-box
make[1]: Entering directory '/home/ereslibre/projects/kubernetes-cluster-vagrant/base-box'
>>> Base box (kubernetes-vagrant) already exists, skipping build
make[1]: Leaving directory '/home/ereslibre/projects/kubernetes-cluster-vagrant/base-box'
vagrant up
Bringing machine 'kubernetes_master' up with 'virtualbox' provider...
Bringing machine 'kubernetes_worker' up with 'virtualbox' provider...

<snip>

>>> kubeconfig written to /home/ereslibre/.kube/config
9.30user 4.82system 2:14.49elapsed 10%CPU (0avgtext+0avgdata 64236maxresident)k
0inputs+584outputs (0major+690067minor)pagefaults 0swaps
```

After a couple of minutes or so, you can execute from your host directly:

```
~/p/kubernetes-cluster-vagrant (master) > kubectl get nodes
NAME      STATUS    ROLES     AGE       VERSION
master    Ready     master    74s       v1.14.0-alpha.0.756+51453a31317118
worker    Ready     <none>    16s       v1.14.0-alpha.0.756+51453a31317118
```

This will use all the versions of packages and components present in the base
box, what might not be what you are looking for, since your local changes to
Kubernetes weren't pushed, because on the `make` command we didn't specify
`PACKAGES`, `IMAGES` or `MANIFESTS` environment variables, so the ones from the
base box were used.

Everything that you want to deploy to the cluster can be controlled by
environment variables. The currently supported ones are:

* `PACKAGES`
  * `cri-tools`
  * `kubeadm`
  * `kubectl`
  * `kubelet`
  * `kubernetes-cni`
* `IMAGES`
  * `kube-apiserver`
  * `kube-controller-manager`
  * `kube-proxy`
  * `kube-scheduler`
* `MANIFESTS`
  * `flannel`

It's possible to start the cluster and make it use your recently built component
versions. You just need to provide what `PACKAGES`, `IMAGES` or `MANIFESTS`
you want to be deployed when the cluster is created. For example:

```
~/p/kubernetes-cluster-vagrant (master) > PROFILE=bootstrap/1-master-1-worker PACKAGES=kubeadm,kubelet make
```

In this case, it doesn't matter what versions of `kubeadm` and `kubelet`
existed on the base box, the ones that you built on the host will be transferred
to the different cluster machines and installed as soon as the machines are up.

In the case that you are using a profile that is bootstrapping the cluster
automatically, this bootstrap will happen after the new packages have overriden
the old ones.

## Deploying changes to the cluster while it's running

`vagrant provision` can be used to deploy different things to the cluster. As
with a regular `vagrant up` (or `make`), you can also use `PACKAGES`,
`IMAGES` and `MANIFESTS` environment variables at will, to control what to
deploy.

```
~/p/kubernetes-cluster-vagrant (master) > PROFILE=bootstrap/1-master-1-worker PACKAGES=kubeadm,kubelet IMAGES=kube-apiserver,kube-scheduler vagrant provision
==> kubernetes_master: Running provisioner: file...
==> kubernetes_master: Running provisioner: file...
==> kubernetes_master: Running provisioner: file...
==> kubernetes_master: Running provisioner: file...
==> kubernetes_master: Running provisioner: shell...
    kubernetes_master: Running: inline script
    kubernetes_master: (Reading database ... 60170 files and directories currently installed.)
    kubernetes_master: Preparing to unpack .../vagrant/kubernetes/kubeadm.deb ...
    kubernetes_master: Unpacking kubeadm (1.14.0~alpha.0.756+51453a31317118) over (1.14.0~alpha.0.756+51453a31317118) ...
    kubernetes_master: Setting up kubeadm (1.14.0~alpha.0.756+51453a31317118) ...
    kubernetes_master: (Reading database ... 60170 files and directories currently installed.)
    kubernetes_master: Preparing to unpack .../vagrant/kubernetes/kubelet.deb ...
    kubernetes_master: Unpacking kubelet (1.14.0~alpha.0.756+51453a31317118) over (1.14.0~alpha.0.756+51453a31317118) ...
    kubernetes_master: Setting up kubelet (1.14.0~alpha.0.756+51453a31317118) ...
    kubernetes_master: Loaded image: k8s.gcr.io/kube-apiserver:v1.14.0-alpha.0.756_51453a31317118
    kubernetes_master: Loaded image: k8s.gcr.io/kube-scheduler:v1.14.0-alpha.0.756_51453a31317118
==> kubernetes_worker: Running provisioner: file...
==> kubernetes_worker: Running provisioner: file...
==> kubernetes_worker: Running provisioner: file...
==> kubernetes_worker: Running provisioner: file...
==> kubernetes_worker: Running provisioner: shell...
    kubernetes_worker: Running: inline script
    kubernetes_worker: (Reading database ... 60170 files and directories currently installed.)
    kubernetes_worker: Preparing to unpack .../vagrant/kubernetes/kubeadm.deb ...
    kubernetes_worker: Unpacking kubeadm (1.14.0~alpha.0.756+51453a31317118) over (1.14.0~alpha.0.756+51453a31317118) ...
    kubernetes_worker: Setting up kubeadm (1.14.0~alpha.0.756+51453a31317118) ...
    kubernetes_worker: (Reading database ... 60170 files and directories currently installed.)
    kubernetes_worker: Preparing to unpack .../vagrant/kubernetes/kubelet.deb ...
    kubernetes_worker: Unpacking kubelet (1.14.0~alpha.0.756+51453a31317118) over (1.14.0~alpha.0.756+51453a31317118) ...
    kubernetes_worker: Setting up kubelet (1.14.0~alpha.0.756+51453a31317118) ...
    kubernetes_worker: Loaded image: k8s.gcr.io/kube-apiserver:v1.14.0-alpha.0.756_51453a31317118
    kubernetes_worker: Loaded image: k8s.gcr.io/kube-scheduler:v1.14.0-alpha.0.756_51453a31317118
```

## HA deployments (multi master)

Multi master deployments are supported, and are as simple as setting the correct
profile. Example:

```
~/p/kubernetes-cluster-vagrant (master) > time PROFILE=bootstrap/3-masters-1-worker make

<snip>

>>> kubeconfig written to /home/ereslibre/.kube/config
21.03user 11.47system 5:31.05elapsed 9%CPU (0avgtext+0avgdata 76264maxresident)k
0inputs+1320outputs (0major+1679954minor)pagefaults 0swaps
```

From your host, now you can run:

```
~/p/kubernetes-cluster-vagrant (master) > kubectl get nodes
NAME      STATUS    ROLES     AGE       VERSION
master1   Ready     master    3m46s     v1.14.0-alpha.0.756+51453a31317118
master2   Ready     master    2m40s     v1.14.0-alpha.0.756+51453a31317118
master3   Ready     master    92s       v1.14.0-alpha.0.756+51453a31317118
worker    Ready     <none>    26s       v1.14.0-alpha.0.756+51453a31317118
```

```
~/p/kubernetes-cluster-vagrant (master) > kubectl get pods --all-namespaces -o wide
NAMESPACE     NAME                              READY     STATUS    RESTARTS   AGE       IP           NODE      NOMINATED NODE   READINESS GATES
kube-system   coredns-86c58d9df4-gj78t          1/1       Running   0          3m23s     10.244.0.2   master1   <none>           <none>
kube-system   coredns-86c58d9df4-vvdm6          1/1       Running   0          3m23s     10.244.0.3   master1   <none>           <none>
kube-system   etcd-master1                      1/1       Running   0          2m49s     10.0.2.15    master1   <none>           <none>
kube-system   etcd-master2                      1/1       Running   0          2m38s     10.0.2.15    master2   <none>           <none>
kube-system   etcd-master3                      1/1       Running   2          84s       10.0.2.15    master3   <none>           <none>
kube-system   kube-apiserver-master1            1/1       Running   0          2m34s     10.0.2.15    master1   <none>           <none>
kube-system   kube-apiserver-master2            1/1       Running   0          2m38s     10.0.2.15    master2   <none>           <none>
kube-system   kube-apiserver-master3            1/1       Running   1          87s       10.0.2.15    master3   <none>           <none>
kube-system   kube-controller-manager-master1   1/1       Running   0          2m38s     10.0.2.15    master1   <none>           <none>
kube-system   kube-controller-manager-master2   1/1       Running   0          2m38s     10.0.2.15    master2   <none>           <none>
kube-system   kube-controller-manager-master3   1/1       Running   0          85s       10.0.2.15    master3   <none>           <none>
kube-system   kube-flannel-ds-amd64-7gv4p       1/1       Running   0          91s       10.0.2.15    master3   <none>           <none>
kube-system   kube-flannel-ds-amd64-gj78r       1/1       Running   0          3m23s     10.0.2.15    master1   <none>           <none>
kube-system   kube-flannel-ds-amd64-ljjjk       1/1       Running   1          2m38s     10.0.2.15    master2   <none>           <none>
kube-system   kube-flannel-ds-amd64-vc9mn       1/1       Running   2          26s       10.0.2.15    worker    <none>           <none>
kube-system   kube-proxy-6ffmq                  1/1       Running   0          91s       10.0.2.15    master3   <none>           <none>
kube-system   kube-proxy-9wh8b                  1/1       Running   0          3m23s     10.0.2.15    master1   <none>           <none>
kube-system   kube-proxy-p9hlr                  1/1       Running   0          2m38s     10.0.2.15    master2   <none>           <none>
kube-system   kube-proxy-wm65p                  1/1       Running   0          26s       10.0.2.15    worker    <none>           <none>
kube-system   kube-scheduler-master1            1/1       Running   0          2m43s     10.0.2.15    master1   <none>           <none>
kube-system   kube-scheduler-master2            1/1       Running   0          2m38s     10.0.2.15    master2   <none>           <none>
kube-system   kube-scheduler-master3            1/1       Running   0          86s       10.0.2.15    master3   <none>           <none>
```

A load balancer (haproxy) will be created, what will be the entry point for all
master node apiservers. The `kubeconfig` file that will get generated in your
`$HOME/.kube/config` will include the reference to this load balancer IP
address.

## Destroying the cluster

You can destroy the cluster by pointing to the profile using the `PROFILE`
environment variable and calling to `make clean`. This will also clean up your
`~/.kube` folder on the host.

```
~/p/kubernetes-cluster-vagrant (master) > PROFILE=bootstrap/1-master-1-worker make clean
vagrant destroy -f
==> kubernetes_worker: Forcing shutdown of VM...
==> kubernetes_worker: Destroying VM and associated drives...
==> kubernetes_master: Forcing shutdown of VM...
==> kubernetes_master: Destroying VM and associated drives...
```

## License

```
kubernetes-cluster-vagrant
Copyright (C) 2018 Rafael Fernández López <ereslibre@ereslibre.es>

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program.  If not, see <https://www.gnu.org/licenses/>.
```
