# Variables

```
MAJOR_VERSION=<os_major_version>
```

```
REDHAT_USERNAME=<redhat_username>
```

```
REDHAT_PASSWORD=<redhat_password>
```

```
POOL_ID=<pool id of your subscription>
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

> *Example:*
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
convert2rhel -h
```

#

```
convert2rhel --username {{ REDHAT_USERNAME }} --password {{ REDHAT_PASSWORD }} --pool {{ POOL_ID }}
```

```
reboot
```
