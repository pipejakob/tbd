# tbd - a kubeadm bootstrapper

## What?

tbd is a proposed tool to handle installing kubeadm and its dependencies. The
desire is to create a single, self-contained executable that is capable of
bootstrapping a host to the point where running `kubeadm init` or `kubeadm join`
will be successful.

tbd does not exist yet, but I'm hoping to vet its usefulness and prototype it
soon.

## Why?

kubeadm is a tool that makes installing a Kubernetes cluster seemingly trivial.

One of the problems of kubeadm is that it promises that cluster initialization
is as simple as:

    # On the master
    $ kubeadm init

    # On each node
    $ kubeadm join --token <TOKEN> <MASTER>

However, those simple steps omit the fact that in order to be able to run
`kubeadm init`, you first have to:

  1. install an appropriate version of kubeadm
  2. install a compatible container runtime like Docker
  3. install a compatible version of kubelet
  4. run kubelet with the correct configuration to use the appropriate container
     runtime

On a Debian-based OS, this might look like:

    $ cat <<EOF > /etc/apt/sources.list.d/k8s.list
    deb [arch=amd64] https://apt.dockerproject.org/repo ubuntu-xenial main
    EOF
    $ apt-key adv --keyserver hkp://keyserver.ubuntu.com --recv-keys F76221572C52609D
    $ apt-get update
    $ apt-get install -y apt-transport-https
    $ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
    $ cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
    deb http://apt.kubernetes.io/ kubernetes-xenial main
    EOF
    $ apt-get update
    $ apt-get install -y docker-engine=1.12.0-0~xenial
    $ apt-get install -y kubelet kubeadm
    $ systemctl enable docker || true
    $ systemctl start docker || true
    $ systemctl enable kubelet || true
    $ systemctl start kubelet || true

This also hides the step of configuring kubelet, because the `.deb` packages
perform that step [automatically
now](https://github.com/kubernetes/release/blob/7b25ad5dfc52e57c9905338b2abac3d13b896d23/debian/xenial/kubeadm/channel/stable/etc/systemd/system/kubelet.service.d/10-kubeadm.conf).
If not using a system package, one still has to figure out what flags should be
passed to it, and potentially how to keep it up and running if not using
systemd.

On Redhat/CentOS, the steps would be analogous but different, using `yum`
commands instead of `apt-get`.

On any other OS, the steps would involve downloading the Kubernetes release
tarball and extracting binary files into appropriate locations.

These instructions get out of hand quick, especially if you want to install
Kubernetes in an OS-agnostic way and need to maintain many different boot
scripts (like [kubicorn
does](https://github.com/kris-nova/kubicorn/tree/58ccdfb5cd78ee67cc6146179bfa228707a20ef8/bootstrap)).

## How?

tbd aims to be a drop-in replacement for kubeadm. The simplest invocation is:

    $ tbd

which attempts to autodetect the environment you're running on, figure out what
you already have installed, and install any dependencies you're missing at the
newest versions supported by the tool.

But that's not all: you can also use `tbd --` as a drop-in replacement for
`kubeadm`:

    # Bootstrap all dependencies, and then run "kubeadm init"
    $ tbd -- init

    # Bootstrap all dependencies, and then run "kubeadm join ..."
    $ tbd -- join --token 123456.1234567890123456 https://localhost:6443

Why the `--`? Because tbd will also accept some arguments itself, and the `--`
delimiter signals that any remaining arguments should be passed directly to
kubeadm unaltered once it's installed.

## Versioning

What if you don't want the newest version of kubeadm? What if you want to
override the default, autodetected method of installation? tbd accepts some
arguments to fine-tune exactly how your system gets bootstrapped:

    $ tbd --kubeadm-version 1.7.2 -- init

Is there a critical bug in kubelet 1.7.2? Override it:

    $ tbd --kubeadm-version 1.7.2 --kubelet-version 1.7.1 -- init

Are you on an Ubuntu system, but would prefer to download static binaries
instead of using apt?

    $ tbd --install-method release -- init

What if you want to use a contiuous integration (CI) build?

    $ tbd --kubeadm-version gs://kubernetes-release-dev/ci/v1.8.0-alpha.2.899+2c624e590f5670 -- init

What if you want to test the latest stable Kubernetes 1.6 control plane using
the latest kubeadm 1.7.x release?

    $ tbd --kubeadm-version stable-1.7 -- init --kubernetes-version latest-1.7

The naming and semantics of these flags aren't set in stone, but they are meant
to demonstrate the kind of functionality and power meant to be provided by tbd.

## tbd?

The name "tbd" is a deliberate placeholder until I come up with a better name.
Candidate names considered were things like "z2k" or "ztok" to mean "zero to
kubeadm-ready," but both fail the Google search test of uniqueness. I fully
intend to come up with a better name and perform a global find-and-replace on
this repository.
