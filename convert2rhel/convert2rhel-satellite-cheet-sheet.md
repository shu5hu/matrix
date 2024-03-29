# Variables

```
MAJOR_VERSION=<os_major_version>
```

```
SATELLITE_FQDN=<satellite_full_hostname>
```

```
ACTIVATION_KEY=<activation_key>
```

```
ORGANIZATION_ID=<organization_name>
```

# Convertion proccess

```
yum update
```

* *You must first take your system to the last update before the proccess*

Change the the kernel in the next reboot

```
grubby --set-default /boot/vmlinuz-`rpm -q --qf "%{BUILDTIME}\t%{EVR}.%{ARCH}\n" kernel | sort -nr | head -1 | cut -f2`
```

```
reboot
```

#

* *For the next steps your system must be connected, in case is not, download this files and bring them into your ristricted environment*

```
curl -o /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release https://www.redhat.com/security/data/fd431d51.txt
```

*DEBUG: `cat /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release`*

```
curl --create-dirs -o /etc/rhsm/ca/redhat-uep.pem https://ftp.redhat.com/redhat/convert2rhel/redhat-uep.pem
```

*DEBUG: `cat /etc/rhsm/ca/redhat-uep.pem`*

```
curl -o /etc/yum.repos.d/convert2rhel.repo https://ftp.redhat.com/redhat/convert2rhel/${MAJOR_VERSION}/convert2rhel.repo
```

*DEBUG: `cat /etc/yum.repos.d/convert2rhel.repo`*

> *EXAMPLE:*
>  ```
>  [convert2rhel-for-rhel-7-rpms]
>  name=Convert2RHEL for OS 7
>  baseurl=https://cdn.redhat.com/content/public/convert2rhel/7/x86_64/os/
>  enabled=1
>  gpgcheck=1
>  sslcacert=/etc/rhsm/ca/redhat-uep.pem
>  gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
>  ```

```
yum -y install convert2rhel
```

```
curl --insecure --output /usr/share/convert2rhel/subscription-manager/katello-ca-consumer-latest.noarch.rpm https://${SATELLITE_FQDN}/pub/katello-ca-consumer-latest.noarch.rpm
```

*DEBUG: `ls /usr/share/convert2rhel/subscription-manager/`*

```
convert2rhel -h
```

```
vi activiation_key.ini
```

> ```
> [subscription_manager]
> activation_key = ${ACTIVATION_KEY}

```
chmod 0600 activiation_key.ini
```

#

```
convert2rhel --org ${ORGANIZATION_ID} --config-file activiation_key.ini
```

```
reboot
```
