# Vagrant > Ansible > Openshift

These instructions are based on https://docs.openshift.com/container-platform/4.4/installing/installing_bare_metal/installing-bare-metal.html

http://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.4/latest/

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



Start the first vagrant server that will become the Loadbalancer and DHCP/DNS
server. That should build and get provisioned by ansible:

    vagrant up lb

<!--
- https://traefik.192.168.100.2.xip.io:8443/dashboard/

Username:password is test:test or test1:test1
-->


When the server is finidhed you should be able to get to the HAProxy stats page.

Username: test
Password: test

- http://192.168.100.2:8404/stats


Start a web server locally with `./webroot` as the root directory. The simplest
way is to use the webserver built into python. This web server serves the ipxe
configs and all the installation files.

    python -m http.server --directory ./webroot 8000

Make sure that the virtual machines can access the web server through any
firewalls. On fedora use **firewall-cmd**

    sudo firewall-cmd --zone=libvirt --add-port=8000/tcp


Start the bootstrap server. This should ipxe boot over http.

    vagrant up bootstrap

After a short while you should see the following in the weblogs

    Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
    192.168.100.5 - - [22/Jun/2020 20:38:48] "GET /bootstrap.ipxe HTTP/1.1" 200 -
    192.168.100.5 - - [22/Jun/2020 20:38:48] "GET /boot.ipxe.cfg HTTP/1.1" 200 -
    192.168.100.5 - - [22/Jun/2020 20:38:48] "GET /images/rhcos-4.4.3-x86_64-installer-kernel-x86_64 HTTP/1.1" 200 -
    192.168.100.5 - - [22/Jun/2020 20:38:48] "GET /images/rhcos-4.4.3-x86_64-installer-initramfs.x86_64.img HTTP/1.1" 200 -
    192.168.100.5 - - [22/Jun/2020 20:39:03] "HEAD /images/rhcos-4.4.3-x86_64-metal.x86_64.raw.gz HTTP/1.1" 200 -
    192.168.100.5 - - [22/Jun/2020 20:39:03] "GET /os_ignition/bootstrap.ign HTTP/1.1" 200 -
    192.168.100.5 - - [22/Jun/2020 20:39:03] "GET /images/rhcos-4.4.3-x86_64-metal.x86_64.raw.gz HTTP/1.1" 200 -

You can follow along using virt-manager and by following the lb server logs.

    vagrant ssh lb
    sudo su -
    journalctl -f

You can examine if the required services are available by looking at the load
balancer stats page. Make sure that the `bootstrap` server is available in both
the `kubernetes_api` and `machine_config` backends. They should go green.



When it has finished installing you can ssh to the machine using the ssh key
created earlier

    ssh -i ssh_key/id_ed25519 core@192.168.100.5

    # to view the progress logs
    journalctl -b -f -u bootkube.service


Start the rest of the machines


    vagrant up /cp[0-9]/


    vagrant up /worker[0-9]/



Wait a while and run the following command. It will exit when the cluster is
ready to have the bootstrap server removed.

    $ openshift-install --dir=webroot/os_ignition wait-for bootstrap-complete --log-level=debug
    DEBUG OpenShift Installer 4.4.6
    DEBUG Built from commit 99e1dc030910ccd241e5d2563f27725c0d3117b0
    INFO Waiting up to 20m0s for the Kubernetes API at https://api.kube1.vm.test:6443...
    INFO API v1.17.1+f63db30 up
    INFO Waiting up to 40m0s for bootstrapping to complete...
    DEBUG Bootstrap status: complete
    INFO It is now safe to remove the bootstrap resources


You can also follow along the logs on the bootstrap server. After what feels
like a year the logs on bootstrap will have something like this appear.

    [core@bootstrap ~]$ journalctl -b -f -u bootkube.service

    Jun 22 21:20:58 bootstrap bootkube.sh[7751]: All self-hosted control plane components successfully started
    Jun 22 21:20:59 bootstrap bootkube.sh[7751]: Sending bootstrap-success event.Waiting for remaining assets to be created.
    # a whole lot more crap
    Jun 22 21:22:36 bootstrap bootkube.sh[7751]: bootkube.service complete


Destroy the bootstrap server

    vagrant destroy bootstrap

Looking in the HAProxy page http://192.168.100.2:8404/stats you will see that
the masters have all gone green.

## Access the cluster with `co`

    export KUBECONFIG=${PWD}/webroot/os_ignition/auth/kubeconfig


    $ oc get nodes
    NAME   STATUS   ROLES    AGE   VERSION
    cp0    Ready    master   26m   v1.17.1
    cp1    Ready    master   25m   v1.17.1
    cp2    Ready    master   25m   v1.17.1


I dont think the workers join unless you have signed their certificates. The
following lists trhe certificates.

    $ oc get csr
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







## Notes

Seem to need to run the following on the machines to get them to work

    sudo systemctl restart systemd-udev-settle.service


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



consoleBaseAddress: https://console-openshift-console.apps.kube1.vm.test
alertmanagerPublicURL: https://alertmanager-main-openshift-monitoring.apps.kube1.vm.test
grafanaPublicURL: https://grafana-openshift-monitoring.apps.kube1.vm.test
prometheusPublicURL: https://prometheus-k8s-openshift-monitoring.apps.kube1.vm.test
thanosPublicURL: https://thanos-querier-openshift-monitoring.apps.kube1.vm.test


https://servicesblog.redhat.com/2019/07/11/installing-openshift-4-1-using-libvirt-and-kvm/
