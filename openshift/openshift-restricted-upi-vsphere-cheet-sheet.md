# Contents

- [Red Hat minimal Quay registry](#Red-Hat-minimal-Quay-registry)
- [Pull OpenShift installation images](#Pull-OpenShift-installation-images)
- [Push images to the local registry](#Push-images-to-the-local-registry)
- [Installation proccess](#Installation-proccess)

</br>

# Prerequisites - 

* [!] This installation required load balancer
 
###### Install Load Balancer in case you don't have one allready 

*install simple load balancer to use on linux machines*

```
yum install haproxy
```

*edit the configuration file to fits your DNS names*

```
vim /etc/haproxy/haproxy.cfg
```

  ```
  global
   log         127.0.0.1 local2
   pidfile     /var/run/haproxy.pid
   maxconn     4000
   daemon
  defaults
   mode                    http
   log                     global
   option                  dontlognull
   option http-server-close
   option                  redispatch
   retries                 3
   timeout http-request    10s
   timeout queue           1m
   timeout connect         10s
   timeout client          1m
   timeout server          1m
   timeout http-keep-alive 10s
   timeout check           10s
   maxconn                 3000
  frontend stats
   bind *:1936
   mode            http
   log             global
   maxconn 10
   stats enable
   stats hide-version
   stats refresh 30s
   stats show-node
   stats show-desc Stats for ocp4 cluster
   stats auth admin:ocp4
   stats uri /stats
  listen api-server-6443
   bind *:6443
   mode tcp
   server bootstrap bootstrap.<cluster name.domain name>:6443 check inter 1s backup
   server master0 master0.<cluster name.domain name>:6443 check inter 1s
   server master1 master1.<cluster name.domain name>:6443 check inter 1s
   server master2 master2.<cluster name.domain name>:6443 check inter 1s
  listen machine-config-server-22623
   bind *:22623
   mode tcp
   server bootstrap bootstrap.<cluster name.domain name>:22623 check inter 1s backup
   server master0 master0.<cluster name.domain name>:22623 check inter 1s
   server master1 master1.<cluster name.domain name>:22623 check inter 1s
   server master2 master2.<cluster name.domain name>:22623 check inter 1s
  listen ingress-router-443
   bind *:443
   mode tcp
   balance source
   server worker0 worker0.<cluster name.domain name>:443 check inter 1s
   server worker1 worker1.<cluster name.domain name>:443 check inter 1s
  listen ingress-router-80
   bind *:80
   mode tcp
   balance source
   server worker0 worker0.<cluster name.domain name>:80 check inter 1s
   server worker1 worker1.<cluster name.domain name>:80 check inter 1s
  ```

* *vim replace: `:` -> `%s/<cluster name.domain name>/<you cluster fqdn>/g`*

*start and enable the service after the config file is ready*

```
systemctl enable-now haproxy.service
```

</br>

# Red Hat minimal Quay registry

###### [!] This registry must have network connection with the bastion machine in the lan 

*Download and install a local, minimal single instance deployment of Red Hat Quay to aid bootstrapping the first disconnected cluster*

https://console.redhat.com/openshift/downloads#tool-mirror-registry

*extract the files from the tar file to your location*

```
tar -xvzf /path/to/mirror-registry.tar.gz -C .
```

> execution-environment.tar </br>
> image-archive.tar </br>
> mirror-registry </br>

###### Create self-signed certificate for the registry login 

```
mkdir ~/openssl
```

```
cd ~/openssl/
```

```
vim ca.cnf
```

  ```
  [ req ]
  distinguished_name = req_distinguished_name
  policy             = policy_anything
  x509_extensions     = v3_ca

  [ req_distinguished_name ]
  commonName                      = Common Name (eg, your name or your server hostname) ## Print this message

  [ v3_ca ]
  subjectKeyIdentifier = hash
  authorityKeyIdentifier = keyid:always,issuer
  basicConstraints = critical,CA:true
  ```

```
openssl genrsa -out ca.key 4096
```

```
openssl req -new -x509 -days 3650 -config ca.cnf -key ca.key -out ca.crt
```

```
vim server.cnf
```

*[!] You **must** add your server FQDN and server IP under [ alt_names ] category*  

  ```
  [ req ]
  distinguished_name  = req_distinguished_name
  policy              = policy_anything
  x509_extensions     = ext
  req_extensions      = v3_req

  [ req_distinguished_name ]
  commonName                      = Common Name (eg, your name or your server hostname)

  [ ext ]
  subjectKeyIdentifier    = hash
  authorityKeyIdentifier  = keyid,issuer
  basicConstraints        = CA:FALSE
  extendedKeyUsage        = serverAuth
  keyUsage                = nonRepudiation, digitalSignature, keyEncipherment, dataEncipherment
  subjectAltName          = @alt_names

  [ v3_req ]
  basicConstraints        = CA:FALSE
  extendedKeyUsage        = serverAuth
  keyUsage                = nonRepudiation, digitalSignature, keyEncipherment, dataEncipherment
  subjectAltName          = @alt_names

  [ alt_names ]
  IP.1 = 1.1.1.1
  DNS.1 = <your.server.com>
  DNS.2 = <your>
  ```
```
openssl genrsa -out server.key 4096
```

```
openssl req -config server.cnf -new -key server.key -out server.csr
```

```
openssl x509 -req -days 365 -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -extensions ext -extfile server.cnf
```

```
cp <path/to/ca_certificate> /etc/pki/ca-trust/source/anchors/
```

```
update-ca-trust
```

*create directory for the mirror-registry*

```
mkdir /quay
```

*run the mirror registry installation with the server self-signed certificate*

```
./mirror-registry install --quayHostname $(hostname -f) --quayRoot /quay/ --sslCert </path/to/server_cert> --sslKey </path/to/server_key>
```

* *[!] The installation will generate login credentials for the mirror-registry, **KEEP THEM***

  *`Example: { init, 6Ioty2XCw3H0Tk1549qpfsB7DlGRVj8g }`*

</br>

# Pull OpenShift installation images

* *[!] This proccess requires internet access*

*download oc client cli*

[https://access.redhat.com/downloads/content/](https://access.redhat.com/downloads/content/290/ver=4.12/rhel---9/4.12/x86_64/product-software)

*extract the binary files*

```
tar -xvzf oc-<version>-linux.tar.gz
```

*install oc client*

```
install oc /usr/local/bin
```

*check oc version*

```
oc version
```

*create OCP_RELEASE variable with the oc version as value*

```
OCP_RELEASE=<client version from the `oc version` stdout>
```

*download your pull-secret*

https://console.redhat.com/openshift/install/pull-secret

*convert your pull-secret file to JSON format*

```
cat ./pull-secret.txt | jq . > ~/pull-secret.json
```

*create variables to pull the relevent images for the installation*

```
LOCAL_REGISTRY='<local_registry_host_name>:<local_registry_host_port>'
```

```
LOCAL_REPOSITORY='ocp4/openshift4'
```

```
PRODUCT_REPO='openshift-release-dev'
```

```
LOCAL_SECRET_JSON='<path_to_pull_secret>'
```

```
RELEASE_NAME="ocp-release"
```

```
ARCHITECTURE=`uname -m`
```

*create directory for the images*

```
mkdir mirror-images
```

*create variable to use it in the pull command, [!] **the path must be absolute path***

```
REMOVABLE_MEDIA_PATH='/path/to/mirror-images/'
```

*dry run the pull command to generate the imageContentSources from the output, use it later in the install-config file*

```
oc adm release mirror -a ${LOCAL_SECRET_JSON} --from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE} --to=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} --to-release-image=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}-${ARCHITECTURE} --dry-run
```

Example: [!] store the output of the last command to use latter in the install-config file
     
    imageContentSources:
    - mirrors:
      - <your.registry.server>:8443/ocp4/openshift4
      source: quay.io/openshift-release-dev/ocp-release
    - mirrors:
      - <your.registry.server>:8443/ocp4/openshift4
      source: quay.io/openshift-release-dev/ocp-v4.0-art-dev


*pull the images from Red-Hat to the local directory to your local machine*

```
oc adm release mirror -a ${LOCAL_SECRET_JSON} --to-dir=${REMOVABLE_MEDIA_PATH}/mirror quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE}
```

</br>

# Push images to the local registry

* *Bring the OC cleint from the connected server*

```
tar -xvzf oc-<version>-linux.tar.gz .
```

*install oc client*

```
install oc /usr/local/bin
```

*check oc version*

```
oc version
```

*create OCP_RELEASE variable with the oc version as value*

```
OCP_RELEASE=<client version from the `oc version` stdout>
```

*login to your registry*

```
podman login -u init -p <password>
```

*create pull-scret with with the credentials of your registry*

```
cat $XDG_RUNTIME_DIR/containers/auth.json > ~/pull-secret.json
```

*create variables to push the relevant images to your registry for the installtion*

```
LOCAL_REGISTRY='<local_registry_host_name>:<local_registry_host_port>'
```

```
LOCAL_REPOSITORY='ocp4/openshift4'
```

```
PRODUCT_REPO='openshift-release-dev'
```

```
LOCAL_SECRET_JSON='<path_to_pull_secret>'
```

```
RELEASE_NAME="ocp-release"
```

```
ARCHITECTURE=`uname -m`
```

*create directory for the images*

```
mkdir mirror-images
```

*create variable to use it in the push command, [!] the path must be absolute path*

```
REMOVABLE_MEDIA_PATH='/path/to/mirror-images/'
```

*push the images to your local registry*

```
oc image mirror -a ${LOCAL_SECRET_JSON} --from-dir=${REMOVABLE_MEDIA_PATH}/mirror "file://openshift/release:${OCP_RELEASE}*" ${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} 
```

</br>

# Installation proccess

*downlowd the openshift installer with the same version as the oc client*

[https://access.redhat.com/downloads/content/](https://access.redhat.com/downloads/content/290/ver=4.12/rhel---9/4.12/x86_64/product-software)

*extract the binary files to the designated location*

```
tar -xvzf tar/openshift-install-linux.tar.gz
```

*check the version of the insatll tool,[!] the version **must** be the same as the oc client tool*

```
./openshift-install version
```

*check the DNS resolve for your machines*

```
dig +noall +answer @<nameserver_ip> api.<cluster_name>.<base_domain> 
```

```
dig +noall +answer @<nameserver_ip> console-openshift-console.apps.<cluster_name>.<base_domain>
```

```
dig +noall +answer @<nameserver_ip> -x <openshift machines ip>
```

*generate ssh key*

```
ssh-keygen
```

*check the public key*

```
cat <path>/<file_name>.pub
```

*start the ssh agent*

```
eval "$(ssh-agent -s)"
```

*add your praivet key to the ssh agent*

```
ssh-add <path>/<private key>
```

*create installation directory*

```
mkdir <installation_directory>
```

*create variable for the installation directory*

```
export INSTALL_DIR=</path/to/installation_directory>
```

*generate install-config file*

```
./openshift-install create install-config --dir ${INSTALL_DIR}
```

*edit the install-config file to fit your needs*

```
vim install-config.yaml
```

        apiVersion: v1
        baseDomain: example.com 
        compute: 
        - hyperthreading: Enabled 
          name: worker
          replicas: 0 
        controlPlane: 
          hyperthreading: Enabled 
          name: master
          replicas: 3 
        metadata:
          name: test 
        platform:
          vsphere:
            vcenter: <your.vcenter.server> 
            username: <username> 
            password: <password> 
            datacenter: <datacenter> 
            defaultDatastore: <datastore> 
            folder: "/<datacenter_name>/vm/<folder_name>/<subfolder_name>" 
            resourcePool: "/<datacenter_name>/host/<cluster_name>/Resources/<resource_pool_name>" 
            diskType: thin 
        fips: false 
        pullSecret: '{"auths":{"<local_registry>": {"auth": "<credentials>","email": "you@example.com"}}}' 
        sshKey: 'ssh-ed25519 AAAA...' 
        additionalTrustBundle: | 
          -----BEGIN CERTIFICATE-----
          ZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZZ
          -----END CERTIFICATE-----
        imageContentSources: 
        - mirrors:
          - <local_registry>/<local_repository_name>/release
          source: quay.io/openshift-release-dev/ocp-release
        - mirrors:
          - <local_registry>/<local_repository_name>/release
          source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
          
*create copy of the install-config file. [!] **VERY IMPORTENT***

```
cp </path/to/install-config.yaml> <install-config.yaml.bak>
```

*generate the mnifests files*

```
./openshift-install create manifests --dir ${INSTALL_DIR}
```

*get in the insatll directory*

```
cd ${INSTALL_DIR}
```

*delete the master and worker yaml files*

```
rm -f openshift/99_openshift-cluster-api_master-machines-*.yaml openshift/99_openshift-cluster-api_worker-machineset-*.yaml
```

*change <mastersSchedulable:> parameter to - false. this change parameter to prevent the orcestrator to run pods on the master nodes*

```
sed -i 's/true/false/g' manifests/cluster-scheduler-02-config.yml
```

*go back to the directory with the installer tool*

```
cd <path to the directory with the install script>
```

*generate the ignition files*

```
./openshift-install create ignition-configs --dir ${INSTALL_DIR}
``` 

*convert the ignition files to base64 format*

```
base64 -w0 ${INSTALL_DIR}/master.ign > ${INSTALL_DIR}/master.64
```

```
base64 -w0 ${INSTALL_DIR}/worker.ign > ${INSTALL_DIR}/worker.64
```

```
base64 -w0 ${INSTALL_DIR}/bootstrap.ign > ${INSTALL_DIR}/bootstrap.64
```

*get the inventory name from the infraID file and create directory for the cluster with this name*

```
jq -r .infraID ${INSTALL_DIR}/metadata.json 
```

*download the rhcos-vmware.x86_64. [!] this template depending on your infrastructure*

https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/

*deploy the coreOS template to the cluster directory*

* *[!] **Do not start** the coreOS Ttemplate.*

###### Create clones from coreOS template 

*Minimal requirements -*
        
                    CPU   MEMORY    STORAGE
        bootstrap   4     16        100
        master0     4     16        100
        master1     4     16        100
        master2     4     16        100
        worker0     2     8         100
        worker1     2     8         100
        
###### Add the parameters above to the advance config in your machine to load the machine with the ignition files

    guestinfo.afterburn.initrd.network-karg = <ip=<machine ip>::<default gateway>:<prefix>:<hostname>:<nic>:none nameserver=<dns server ip1> nameserver=<dns server ip2> ...> 
    guestinfo.ignition.config.data = <copy of the ignition file content>
    guestinfo.ignition.config.data.encoding = base64
    disk.EnableUUID  = TRUE
    stealclock.enable = TRUE

*run the bootstrap complete command before workers created*

```
./openshift-install wait-for bootstrap-complete --dir ${INSTALL_DIR} --log-level debug
```

*copy the kubconfig file to the designated location*

```
cp <installation directory>/auth/kubeconfig .kube/config
```
***For Losers** - `export KUBECONFIG=${INSTALL_DIR}/auth/kubeconfig`*

*search for nodes request*

```
oc get csr |grep Pending
```

*approve all the nodes requsets*

```
oc get csr |grep -i pending |awk '{print $1}' |while read x;do oc adm certificate approve $x;done
```

* *Run this command many times*

*run this command to wait and pray for the installation tp complete successfully*

```
./openshift-install wait-for install-complete --dir ${INSTALL_DIR} --log-level=debug
```

* Optional:

*disable the operator hub sources from the internet*

```
oc patch OperatorHub cluster --type json -p '[{"op": "add", "path": "/spec/disableAllDefaultSources", "value": true}]'
```

