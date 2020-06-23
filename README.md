# Vagrant > Ansible > Openshift

These instructions are based on https://docs.openshift.com/container-platform/4.4/installing/installing_bare_metal/installing-bare-metal.html


Make some directories to use.

    mkdir -p bin tmp ssh_key
    export PATH=${PWD}/bin/:$PATH

Generating an SSH keypair

    ssh-keygen -o -a 100 -t ed25519 -f ssh_key/id_ed25519  -C "example openshift key"


Obtaining a **Pull Secret** from the following URI and save it as
`./tmp/pull-secret.txt`

- https://cloud.redhat.com/openshift/install/pull-secret

Download the installation file on a local computer.

- https://cloud.redhat.com/openshift/install/metal/user-provisioned


    curl -Lo tmp/openshift-install-linux.tar.gz https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-install-linux.tar.gz

Extract the installation program and put it somewhere on your PATH.


    tar xvf tmp/openshift-install-linux.tar.gz --directory tmp/
    mv tmp/openshift-install bin/

Download and extract the command line tool `oc` in the same place as you put
the installer.

    curl -Lo tmp/openshift-client-linux.tar.gz https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-linux.tar.gz
    tar xvf tmp/openshift-client-linux.tar.gz --directory tmp/
    mv tmp/{oc,kubectl} bin/


Create an installation directory

    mkdir -p webroot/os_ignition

If yuo have used ithe directory before then you need to remove the old files.

    rm -rf webroot/os_ignition/*
    rm -rf webroot/os_ignition/.openshift*

Create an `install-config.yaml` inside the installatin directory and customize
it as per the documentation on https://docs.openshift.com/container-platform/4.4/installing/installing_bare_metal/installing-bare-metal.html#installation-bare-metal-config-yaml_installing-bare-metal
Note that you need to put your pull secret in there.

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
    pullSecret: '$(cat tmp/pull-secret.txt)'
    sshKey: '$(cat ssh_key/id_ed25519.pub)'
    EOF

I think you can add a corporate CA certificate in the insta;l;-config.yaml

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

Creating the Kubernetes manifest and Ignition config files

    openshift-install create manifests --dir=webroot/os_ignition

    sed -i 's/  mastersSchedulable: true/  mastersSchedulable: false/g' webroot/os_ignition/manifests/cluster-scheduler-02-config.yml

    openshift-install create ignition-configs --dir=webroot/os_ignition


XXXX TODO: besides the ignition files this also create the kubernetes
authentication files and thesy shouldnt be uploaded to a web servers.


Download the install images into `webroot/images/`


    (
    cd webroot/images/
    curl -LO https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/latest/latest/rhcos-4.4.3-x86_64-installer-kernel-x86_64
    curl -LO https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/latest/latest/rhcos-4.4.3-x86_64-installer-initramfs.x86_64.img
    curl -LO https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/latest/latest/rhcos-4.4.3-x86_64-metal.x86_64.raw.gz
    )


## The loadbalancer system.



Start the first vagrant system that will provide the Loadbalancer and DHCP/DNS
services. In a production environment this would be a part of your infrastructure.
The following vagrant command should build the *lb* system and provision it using ansible:

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
configs and all the installation files.

    python -m http.server --directory ./webroot 8000

Make sure that the virtual machines can access the web server through any
firewalls. On fedora use **firewall-cmd**

    sudo firewall-cmd --zone=libvirt --add-port=8000/tcp

Start the *bootstrap* system. This should ipxe boot over http.

    vagrant up bootstrap

After a short while you should see the following in the weblogs.

    Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
    192.168.100.5 - - [22/Jun/2020 20:38:48] "GET /bootstrap.ipxe HTTP/1.1" 200 -
    192.168.100.5 - - [22/Jun/2020 20:38:48] "GET /boot.ipxe.cfg HTTP/1.1" 200 -
    192.168.100.5 - - [22/Jun/2020 20:38:48] "GET /images/rhcos-4.4.3-x86_64-installer-kernel-x86_64 HTTP/1.1" 200 -
    192.168.100.5 - - [22/Jun/2020 20:38:48] "GET /images/rhcos-4.4.3-x86_64-installer-initramfs.x86_64.img HTTP/1.1" 200 -
    192.168.100.5 - - [22/Jun/2020 20:39:03] "HEAD /images/rhcos-4.4.3-x86_64-metal.x86_64.raw.gz HTTP/1.1" 200 -
    192.168.100.5 - - [22/Jun/2020 20:39:03] "GET /os_ignition/bootstrap.ign HTTP/1.1" 200 -
    192.168.100.5 - - [22/Jun/2020 20:39:03] "GET /images/rhcos-4.4.3-x86_64-metal.x86_64.raw.gz HTTP/1.1" 200 -


The *bootstrap* system is ready when it becomes available in the Loadbalancer. You
can examine if the required services are available by looking at the load
balancer stats page. Make sure that the *bootstrap* system is available in both
the `kubernetes_api` and `machine_config` backends. They should go green.



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



    $ openshift-install --dir=webroot/os_ignition wait-for bootstrap-complete --log-level=debug
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

    export KUBECONFIG=${PWD}/webroot/os_ignition/auth/kubeconfig


    $ watch -n5 oc get nodes
    NAME   STATUS   ROLES    AGE   VERSION
    cp0    Ready    master   26m   v1.17.1
    cp1    Ready    master   25m   v1.17.1
    cp2    Ready    master   25m   v1.17.1


For a new *worker* system to join the cluster it will need it's certificate
approved.  This can be automated in a cloud provider but for a baremetal cluster
you need to do it by ahand or automate itr in some way. Keep an eye on this
while the workers are building as they have 2 sets of certificates which need
signing.


    $ watch -n5 oc get csr
    NAME        AGE   REQUESTOR                                                                   CONDITION
    csr-2vn2n   34m   system:node:cp2                                                             Approved,Issued
    csr-57kvs   35m   system:node:cp0                                                             Approved,Issued
    csr-8mhbr   34m   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued
    csr-9wlf9   11m   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
    csr-dbrb2   10m   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
    csr-fh87f   10m   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
    csr-h5g2s   35m   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued
    csr-kxxxg   34m   system:node:cp1                                                             Approved,Issued
    csr-tsw2q   34m   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued

You can sign them one at a time but this does it all at once.

    oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve


Watch the cluster operators start up.

    watch -n5 oc get clusteroperators


Eventually you should get everything go to *True* in the **AVAILABLE** column.

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


## Access the web interface




- https://console-openshift-console.apps.kube1.vm.test

To login for the first time you should use the user `kubeadmin` and the password
that was generated when you set up the cluster config files.

    cat webroot/os_ignition/auth/kubeadmin-password




## Notes



htaccesss provider

    htpasswd -c -B -b users.htpasswd test test
    oc create secret generic htpass-secret --from-file=htpasswd=users.htpasswd     -n openshift-config

    cat >  htaccess.yaml
    apiVersion: config.openshift.io/v1
    kind: OAuth
    metadata:
    name: cluster
    spec:
    identityProviders:
    - name: my_htpasswd_provider
        mappingMethod: claim
        type: HTPasswd
        htpasswd:
        fileData:
            name: htpass-secret


    oc apply -f htaccess.yaml

    oc rsh -n openshift-authentication oauth-openshift-667b4c7565-7spwz cat /run/secrets/kubernetes.io/serviceaccount/ca.crt > ingress-ca.crt
    oc login -u test  -p test --certificate-authority=ingress-ca.crt


