# Vagrant > Ansible > OKD4

OKD4 is the upstream community version of Red Hat's OpenShift Container
Platform. This system,  which is based on Kubernetes and containers, enables you
to run you own Platform-as-a-Service on your own hardware or deploy it into a
cloud platform such as Google Cloud or AWS.

These instructions are based on https://docs.okd.io/latest/installing/installing_bare_metal/installing-bare-metal.html#installation-requirements-user-infra_installing-bare-metal

I have built this as a learning experience and potential use as a playground
environment for Systems Engineers who manage OKD4 clusters. At the end you
should end up with a OKD4 cluster with:

- One load-balancer/dhcp/dns system
- Three control-plane systems
- Three worker systems.


## Security

Don't expect this to be secure in any way. There are several thing in here that
I know to be insecure but this is for setting up a test environment. You have
been warned.

## Prerequisites

This walkthrough is based on my working environment.

- Fedora
- Vagrant
- vagrant-libvirt
- Ansible


There is no real dependency on Fedora as long as you have fairly recent versions
of the other items. At the time of writing I was using Fedora 32 but this should
work with minimal modifications on other Linux distributions. You will also need
a lot of RAM as you are going to start up 8 virtual machines so the more RAM the
better. If you less than 64GB then you may want to edit the amount allocated to
each machine type in the `Vagrantfile`

You will also need some way of accessing the dns inside the cluster. This can
easily be done with NetworkManager and dnsmasq, an example of this is below.

    $ cat /etc/NetworkManager/conf.d/00-use-dnsmasq.conf
    # /etc/NetworkManager/conf.d/00-use-dnsmasq.conf
    #
    # This enabled the dnsmasq plugin.
    [main]
    dns=dnsmasq

    $ cat  /etc/NetworkManager/dnsmasq.d/00-config.conf
    server=192.168.1.1
    addn-hosts=/etc/hosts

    $  cat /etc/NetworkManager/dnsmasq.d/00-okd.conf
    server=/kube1.vm.test/192.168.100.2

Remember to restart NetworkManager to pick up the changes.

    systemctl restart NetworkManager.service


# Get the code.

Step one is to clone the code which is available on Github.

    git clone https://github.com/timhughes/vagrant_ansible_okd4
    cd vagrant_ansible_okd4

The working directory for for all the commands is assumed to be the root
directory of the cloned git repository.


## Getting the cli tools and preparing.


Make some directories to use.

    mkdir -p bin tmp ssh_key
    export PATH=${PWD}/bin/:$PATH

Generating an SSH keypair

    ssh-keygen -o -a 100 -t ed25519 -f ssh_key/id_ed25519  -C "example okd key"


Discover the revision of the latest OKD release and set it as a variable we can
use throughout the rest of the process. If you open another terminal you will
need to run this again.

    export OKD_RELEASE=$(curl --silent "https://api.github.com/repos/openshift/okd/releases/latest" | grep -Po '"tag_name": "\K.*?(?=")')

Download the installation file on a local computer.

- https://github.com/openshift/okd/releases
```
wget -c -LP tmp/ https://github.com/openshift/okd/releases/download/${OKD_RELEASE}/openshift-install-linux-${OKD_RELEASE}.tar.gz
```
Extract the installation program and put it somewhere on your PATH.


    tar xvf tmp/openshift-install-linux-${OKD_RELEASE}.tar.gz --directory tmp/
    mv tmp/openshift-install bin/

Download and extract the command line tool `oc` from the same locattion as the
installer and pit it in the same place as you put the installer.

    wget -c -LP tmp/ https://github.com/openshift/okd/releases/download/${OKD_RELEASE}/openshift-client-linux-${OKD_RELEASE}.tar.gz
    tar xvf tmp/openshift-client-linux-${OKD_RELEASE}.tar.gz --directory tmp/
    mv tmp/{oc,kubectl} bin/


Create an installation directory

    mkdir -p webroot/os_ignition

If you have used the directory before then you need to remove the old files.

    rm -rf webroot/os_ignition/*
    rm -rf webroot/os_ignition/.openshift*


## Creating the boot configs.

Create an `install-config.yaml` inside the installation directory and customize
it as per the documentation on
https://docs.okd.io/latest/installing/installing_bare_metal/installing-bare-metal.html#installation-bare-metal-config-yaml_installing-bare-metal
Note that you do not need valid value for `pullSecret` because that is for the
Red Hat supported OCP:

    cat <<-EOF > webroot/os_ignition/install-config.yaml
    apiVersion: v1
    baseDomain: vm.test
    compute:
    - hyperthreading: Enabled
      name: worker
      replicas: 0
    controlPlane:
      hyperthreading: Enabled
      name: master
      replicas: 3
    metadata:
      name: kube1
    networking:
      clusterNetwork:
      - cidr: 10.128.0.0/14
        hostPrefix: 23
      networkType: OpenShiftSDN
      serviceNetwork:
      - 192.168.101.0/24  # different to host network
    platform:
      none: {}
    fips: false
    pullSecret: '{"auths":{"xxxxxxx": {"auth": "xxxxxx","email": "xxxxxx"}}}'
    sshKey: '$(cat ssh_key/id_ed25519.pub)'
    EOF

You can add a corporate CA certificate and proxy servers in the
`install-config.yaml` if required:

    apiVersion: v1
    baseDomain: my.domain.com
    proxy:
        httpProxy: http://<username>:<pswd>@<ip>:<port>
        httpsProxy: http://<username>:<pswd>@<ip>:<port>
        noProxy: example.com
    additionalTrustBundle: |
        -----BEGIN CERTIFICATE-----
        <MY_TRUSTED_CA_CERT>
        -----END CERTIFICATE-----

Creating the Kubernetes manifest and Ignition config files:

    openshift-install create manifests --dir=webroot/os_ignition

    sed -i 's/  mastersSchedulable: true/  mastersSchedulable: false/g' webroot/os_ignition/manifests/cluster-scheduler-02-config.yml

    openshift-install create ignition-configs --dir=webroot/os_ignition


Besides the ignition files this also create the kubernetes
authentication files and they shouldn't be uploaded to a web servers.

    #TODO copy the ignition files to a web server with out the auth files

Download the install images and sig files into `webroot/images/`. You can do
this by hand or use the following command to grab the latest automatically:

    curl -s "https://builds.coreos.fedoraproject.org/streams/stable.json"|jq '.architectures.x86_64.artifacts.metal.formats| .pxe,."raw.xz"|.[].location' | xargs wget -c -LP webroot/images/
    curl -s "https://builds.coreos.fedoraproject.org/streams/stable.json"|jq '.architectures.x86_64.artifacts.metal.formats| .pxe,."raw.xz"|.[].location,.[].signature'| xargs wget -c -LP webroot/images/


Update `webroot/boot.ipxe.cfg` to match the filename and versions you downloaded
you downloaded.

    set okd-kernel fedora-coreos-32.20200629.3.0-live-kernel-x86_64
    set okd-initrd fedora-coreos-32.20200629.3.0-live-initramfs.x86_64.img
    set okd-image fedora-coreos-32.20200629.3.0-metal.x86_64.raw.xz


## The load-balancer system.


Start the first vagrant system that will provide the load-balancer and DHCP/DNS
services. In a production environment this would be a part of your
infrastructure.  The following vagrant command should build the *lb* system and
provision it using Ansible:

    vagrant up lb


The *lb* system is ready when you can access the HAProxy web
interface.

Username: test
Password: test

- http://192.168.100.2:8404/stats

You can access the *lb* system via vagrant:

    vagrant ssh lb


<!--
- https://traefik.192.168.100.2.xip.io:8443/dashboard/

Username:password is test:test or test1:test1
-->



## Building the Bootstrap system

Start a web server locally with `./webroot` as the root directory. The simplest
way is to use the webserver built into python. This web server serves the ipxe
configs and all the installation files. 🔥 the `./webroot/os_ignition/auth`
directory contains files that dont need to be and should never be accessable, I
have just been lazy here since it is a network local to wy workstation.

    python -m http.server --bind 192.168.100.1 --directory ./webroot 8000

Make sure that the virtual machines can access the web server through any
firewalls. On fedora use **firewall-cmd**

    sudo firewall-cmd --zone=libvirt --add-port=8000/tcp

Start the *bootstrap* system. This should ipxe boot over http.

    vagrant up bootstrap

After a short while you should see the following in the weblogs.

    Serving HTTP on 192.168.100.1 port 8000 (http://192.168.100.1:8000/) ...
    192.168.100.5 - - [16/Jul/2020 17:33:05] "GET /bootstrap.ipxe HTTP/1.1" 200 -
    192.168.100.5 - - [16/Jul/2020 17:33:05] "GET /boot.ipxe.cfg HTTP/1.1" 200 -
    192.168.100.5 - - [16/Jul/2020 17:33:05] "GET /images/fedora-coreos-32.20200629.3.0-live-kernel-x86_64 HTTP/1.1" 200 -
    192.168.100.5 - - [16/Jul/2020 17:33:05] "GET /images/fedora-coreos-32.20200629.3.0-live-initramfs.x86_64.img HTTP/1.1" 200 -
    192.168.100.5 - - [16/Jul/2020 17:33:24] "GET /os_ignition/bootstrap.ign HTTP/1.1" 200 -
    192.168.100.5 - - [16/Jul/2020 17:33:25] "GET /images/fedora-coreos-32.20200629.3.0-metal.x86_64.raw.xz.sig HTTP/1.1" 200 -
    192.168.100.5 - - [16/Jul/2020 17:33:25] "GET /images/fedora-coreos-32.20200629.3.0-metal.x86_64.raw.xz HTTP/1.1" 200 -


The *bootstrap* system is ready when it becomes available in the load-balancer
You can examine if the required services are available by looking at the
load-balancer stats page. Make sure that the *bootstrap* system is available in
both the `kubernetes_api` and `machine_config` backends. They should go green.


When it has finished installing you can ssh to the system using the ssh key
created earlier.

    ssh -i ssh_key/id_ed25519 core@192.168.100.5

    # to view the progress logs
    journalctl -b -f -u bootkube.service


## Control Plane systems and Worker systems
Start the rest of the systems. This requires 3 control plane (*cp*) systems and at
least 2 *worker* systems. The following commands will start these up three of
each.


    vagrant up /cp[0-9]/


    vagrant up /worker[0-9]/



You need to remove the *bootstrap* system when it has finished doing the initial
setup of the *cp* systems. The following command will monitor the
bootstrap progress and report when it is complete.



    $ openshift-install --dir=. wait-for bootstrap-complete --log-level=debug
    DEBUG OpenShift Installer 4.4.6
    DEBUG Built from commit 99e1dc030910ccd241e5d2563f27725c0d3117b0
    INFO Waiting up to 20m0s for the Kubernetes API at https://api.kube1.vm.test:6443...
    INFO API v1.17.1+f63db30 up
    INFO Waiting up to 40m0s for bootstrapping to complete...
    DEBUG Bootstrap status: complete
    INFO It is now safe to remove the bootstrap resources


You can also follow along the logs on the *bootstrap* system. After what feels
like a year the logs on *bootstrap* will have something like this appear.

    [core@bootstrap ~]$ journalctl -b -f -u bootkube.service

    Jun 22 21:20:58 bootstrap bootkube.sh[7751]: All self-hosted control plane components successfully started
    Jun 22 21:20:59 bootstrap bootkube.sh[7751]: Sending bootstrap-success event.Waiting for remaining assets to be created.
    # a whole lot more crap
    Jun 22 21:22:36 bootstrap bootkube.sh[7751]: bootkube.service complete


Destroy the *bootstrap* system

    vagrant destroy bootstrap

Looking in the HAProxy page http://192.168.100.2:8404/stats you will see that
the *cp* systems have all gone green and the *bootstrap* system is now red

## Access the cluster with `co`

    export KUBECONFIG=webroot/os_ignition/auth/kubeconfig


    $ watch -n5 oc get nodes
    NAME   STATUS   ROLES    AGE   VERSION
    cp0    Ready    master   26m   v1.17.1
    cp1    Ready    master   25m   v1.17.1
    cp2    Ready    master   25m   v1.17.1


For a new *worker* system to join the cluster it will need it's certificate
approved.  This can be automated in a cloud provider but for a bare-metal
cluster you need to do it by hand or automate it in some way. Keep an eye on
this while the workers are building as they have 2 sets of certificates which
need signing.


    $ oc get csr
    NAME        AGE   SIGNERNAME                                    REQUESTOR                                                                   CONDITION
    csr-6jxjn   36m   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued
    csr-8fztn   10m   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
    csr-8hmrl   10m   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
    csr-8xzzz   36m   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued
    csr-cktk4   36m   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued
    csr-kktkw   36m   kubernetes.io/kubelet-serving                 system:node:cp2                                                             Approved,Issued
    csr-lkxhx   10m   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
    csr-qlhmk   35m   kubernetes.io/kubelet-serving                 system:node:cp0                                                             Approved,Issued
    csr-wgjdd   36m   kubernetes.io/kubelet-serving                 system:node:cp1                                                             Approved,Issued

You can sign them one at a time but this does it all at once.

    $ oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve
    certificatesigningrequest.certificates.k8s.io/csr-8fztn approved
    certificatesigningrequest.certificates.k8s.io/csr-8hmrl approved
    certificatesigningrequest.certificates.k8s.io/csr-lkxhx approved


This will allow the nodes to registyer and they will almost immediatly request a
certificate of their own which will need signing.

    $ oc get csr
    NAME        AGE   SIGNERNAME                                    REQUESTOR                                                                   CONDITION
    csr-6jxjn   37m   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued
    csr-8fztn   11m   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued
    csr-8hmrl   11m   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued
    csr-8xzzz   37m   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued
    csr-cktk4   37m   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued
    csr-fb9qk   34s   kubernetes.io/kubelet-serving                 system:node:worker-192-168-100-228.kube1.vm.test                            Pending
    csr-hjqv6   33s   kubernetes.io/kubelet-serving                 system:node:worker-192-168-100-230.kube1.vm.test                            Pending
    csr-j42cq   34s   kubernetes.io/kubelet-serving                 system:node:worker-192-168-100-238.kube1.vm.test                            Pending
    csr-kktkw   37m   kubernetes.io/kubelet-serving                 system:node:cp2                                                             Approved,Issued
    csr-lkxhx   11m   kubernetes.io/kube-apiserver-client-kubelet   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued
    csr-qlhmk   36m   kubernetes.io/kubelet-serving                 system:node:cp0                                                             Approved,Issued
    csr-wgjdd   37m   kubernetes.io/kubelet-serving                 system:node:cp1                                                             Approved,Issued

Because there are still come pending you will need to sign them as well.

    $ oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve
    certificatesigningrequest.certificates.k8s.io/csr-fb9qk approved
    certificatesigningrequest.certificates.k8s.io/csr-hjqv6 approved
    certificatesigningrequest.certificates.k8s.io/csr-j42cq approved



Watch the cluster operators start up.

    watch -n5 oc get clusteroperators

Eventually, everything go to *True* in the **AVAILABLE** column.

    NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
    authentication                             4.4.6     True        False         False      30m
    cloud-credential                           4.4.6     True        False         False      80m
    cluster-autoscaler                         4.4.6     True        False         False      46m
    console                                    4.4.6     True        False         False      30m
    csi-snapshot-controller                    4.4.6     True        False         False      38m
    dns                                        4.4.6     True        False         False      65m
    etcd                                       4.4.6     True        False         False      65m
    image-registry                             4.4.6     True        False         False      57m
    ingress                                    4.4.6     True        False         False      40m
    insights                                   4.4.6     True        False         False      57m
    kube-apiserver                             4.4.6     True        False         False      64m
    kube-controller-manager                    4.4.6     True        False         False      64m
    kube-scheduler                             4.4.6     True        False         False      64m
    kube-storage-version-migrator              4.4.6     True        False         False      39m
    machine-api                                4.4.6     True        False         False      66m
    machine-config                             4.4.6     True        False         False      34m
    marketplace                                4.4.6     True        False         False      57m
    monitoring                                 4.4.6     True        False         False      29m
    network                                    4.4.6     True        False         False      67m
    node-tuning                                4.4.6     True        False         False      68m
    openshift-apiserver                        4.4.6     True        False         False      40m
    openshift-controller-manager               4.4.6     True        False         False      47m
    openshift-samples                          4.4.6     True        False         False      22m
    operator-lifecycle-manager                 4.4.6     True        False         False      66m
    operator-lifecycle-manager-catalog         4.4.6     True        False         False      66m
    operator-lifecycle-manager-packageserver   4.4.6     True        False         False      51m
    service-ca                                 4.4.6     True        False         False      68m
    service-catalog-apiserver                  4.4.6     True        False         False      68m
    service-catalog-controller-manager         4.4.6     True        False         False      68m
    storage                                    4.4.6     True        False         False      57m



Use the following command to check when the install is completed.

    openshift-install --dir=webroot/os_ignition wait-for install-complete

The output will hang and wait for the installation to complete. Eventually you
should end up with output similar to the following:

    INFO Waiting up to 30m0s for the cluster at https://api.kube1.vm.test:6443 to initialize...
    INFO Waiting up to 10m0s for the openshift-console route to be created...
    INFO Install complete!
    INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/home/timhughes/git/vagrant_ansible_okd4/webroot/os_ignition/auth/kubeconfig'
    INFO Access the OpenShift web-console here: https://console-openshift-console.apps.kube1.vm.test
    INFO Login to the console with user: kubeadmin, password: xxxxx-xxxxx-xxxxx-xxxxx


## Access the web interface.


- https://console-openshift-console.apps.kube1.vm.test

To login for the first time you should use the user `kubeadmin` and the password
shown in the last section. If you have missed that then it is available in the
configs you set up at the beginning.

    cat webroot/os_ignition/auth/kubeadmin-password





