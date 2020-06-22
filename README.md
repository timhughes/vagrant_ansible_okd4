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
    




Download the install images into `webroot/images/`


    (
    cd webroot/images/
    curl -LO https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/latest/latest/rhcos-4.4.3-x86_64-installer-kernel-x86_64
    curl -LO https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/latest/latest/rhcos-4.4.3-x86_64-installer-initramfs.x86_64.img
    curl -LO https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/latest/latest/rhcos-4.4.3-x86_64-metal.x86_64.raw.gz
    )


Add the pxeboot files 

    sudo dnf install -y syslinux-nonlinux
    cp /usr/share/syslinux/{pxelinux.0,ldlinux.c32} webroot/


Start the first vagrant server that will become the Loadbalancer and DHCP/DNS
server. That should build and get provisioned by ansible:

    vagrant up lb

- https://traefik.192.168.100.2.xip.io:8443/dashboard/

Username:password is test:test or test1:test1 

Start a web server locally with `./webroot` as the root directory. The simplest
way is to use the webserver built into python.

    python -m http.server --directory ./webroot 8000

Make sure that the virtual machines can access the web server through any
firewalls. On fedora use **firewall-cmd**

    sudo firewall-cmd --zone=libvirt --add-port=8000/tcp


Start the bootstrap server. This should ipxe boot over http.

    vagrant up bootstrap

You can follow along using virt-manager and by following the logs on the load balancer

After a short while you should see the following in the weblogs

    Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
    192.168.100.5 - - [19/Jun/2020 01:46:08] "GET /pxelinux.0 HTTP/1.1" 200 -
    192.168.100.5 - - [19/Jun/2020 01:46:08] "GET /ldlinux.c32 HTTP/1.1" 200 -
    192.168.100.5 - - [19/Jun/2020 01:46:08] code 404, message File not found
    192.168.100.5 - - [19/Jun/2020 01:46:08] "GET /pxelinux.cfg/01-52-54-00-a8-64-05 HTTP/1.1" 404 -
    192.168.100.5 - - [19/Jun/2020 01:46:08] code 404, message File not found
    192.168.100.5 - - [19/Jun/2020 01:46:08] "GET /pxelinux.cfg/C0A86405 HTTP/1.1" 404 -
    192.168.100.5 - - [19/Jun/2020 01:46:08] code 404, message File not found
    192.168.100.5 - - [19/Jun/2020 01:46:08] "GET /pxelinux.cfg/C0A8640 HTTP/1.1" 404 -
    192.168.100.5 - - [19/Jun/2020 01:46:08] code 404, message File not found
    192.168.100.5 - - [19/Jun/2020 01:46:08] "GET /pxelinux.cfg/C0A864 HTTP/1.1" 404 -
    192.168.100.5 - - [19/Jun/2020 01:46:08] code 404, message File not found
    192.168.100.5 - - [19/Jun/2020 01:46:08] "GET /pxelinux.cfg/C0A86 HTTP/1.1" 404 -
    192.168.100.5 - - [19/Jun/2020 01:46:08] code 404, message File not found
    192.168.100.5 - - [19/Jun/2020 01:46:08] "GET /pxelinux.cfg/C0A8 HTTP/1.1" 404 -
    192.168.100.5 - - [19/Jun/2020 01:46:08] code 404, message File not found
    192.168.100.5 - - [19/Jun/2020 01:46:08] "GET /pxelinux.cfg/C0A HTTP/1.1" 404 -
    192.168.100.5 - - [19/Jun/2020 01:46:08] code 404, message File not found
    192.168.100.5 - - [19/Jun/2020 01:46:08] "GET /pxelinux.cfg/C0 HTTP/1.1" 404 -
    192.168.100.5 - - [19/Jun/2020 01:46:08] code 404, message File not found
    192.168.100.5 - - [19/Jun/2020 01:46:08] "GET /pxelinux.cfg/C HTTP/1.1" 404 -
    192.168.100.5 - - [19/Jun/2020 01:46:08] "GET /pxelinux.cfg/default HTTP/1.1" 200 -
    192.168.100.5 - - [19/Jun/2020 01:46:08] "GET /images/rhcos-4.4.3-x86_64-installer-kernel-x86_64 HTTP/1.1" 200 -
    192.168.100.5 - - [19/Jun/2020 01:46:08] "GET /images/rhcos-4.4.3-x86_64-installer-initramfs.x86_64.img HTTP/1.1" 200 -
    192.168.100.5 - - [19/Jun/2020 01:46:24] "HEAD /images/rhcos-4.4.3-x86_64-metal.x86_64.raw.gz HTTP/1.1" 200 -
    192.168.100.5 - - [19/Jun/2020 01:46:24] "GET /os_ignition/bootstrap.ign HTTP/1.1" 200 -
    192.168.100.5 - - [19/Jun/2020 01:46:24] "GET /images/rhcos-4.4.3-x86_64-metal.x86_64.raw.gz HTTP/1.1" 200 -



When it has finished installing you can ssh to the machine using the ssh key
created earlier

    ssh -i ssh_key/id_ed25519 core@192.168.100.5
    
Start the rest of the machines:w
 
    
    vagrant up /cp[0-9]/
    vagrant up /compute[0-9]/


    openshift-install --dir=webroot/os_ignition wait-for bootstrap-complete --log-level=info


#### Need to regenerate configs with new domain


## Notes



https://servicesblog.redhat.com/2019/07/11/installing-openshift-4-1-using-libvirt-and-kvm/
