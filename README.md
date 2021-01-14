# Instalacion OCP 4.6 en Vmware con metodo Bare Metal
### Creación Vm BASTION (Centos/RedHat 7)
En la misma vamos a tener DNS/DHCP/HAPROXY/WEB
 ```
[root@helper ~]# yum -y install ansible git
[root@helper ~]# git clone https://github.com/RedHatOfficial/ocp4-helpernode
[root@helper ~]# cd ocp4-helpernode
[root@helper ~]# cp docs/examples/vars.yaml .
[root@helper ~]# vim vars.yaml 
 ```
 En el mismo definimos:
 - <clusteid><domain> En nuestro caso es ocp4.lab.local
 - Definimos los parametros de dhcp
 - Definimos las ips con sus respectivas macaddres para los nodos bootstrap, masters y workers


-----------------------------------------------------------------
---
```
disk: vda
helper:
  name: "helper"
  ipaddr: "10.54.108.220"
dns:
  domain: "lab.local"
  clusterid: "ocp4"
  forwarder1: "8.8.8.8"
  forwarder2: "8.8.4.4"
dhcp:
  router: "10.54.108.1"
  bcast: "10.54.108.255"
  netmask: "255.255.255.0"
  poolstart: "10.54.108.221"
  poolend: "10.54.108.229"
  ipid: "10.54.108.0"
  netmaskid: "255.255.255.0"
bootstrap:
  name: "bootstrap"
  ipaddr: "10.54.108.226"
  macaddr: "00:50:56:bd:fd:fc"
masters:
  - name: "master0"
    ipaddr: "10.54.108.221"
    macaddr: "00:50:56:bd:07:c0"
  - name: "master1"
    ipaddr: "10.54.108.222"
    macaddr: "00:50:56:bd:a2:f0"
  - name: "master2"
    ipaddr: "10.54.108.223"
    macaddr: "00:50:56:bd:9c:a4"
workers:
  - name: "worker0"
    ipaddr: "10.54.108.224"
    macaddr: "00:50:56:bd:24:d7"
  - name: "worker1"
    ipaddr: "10.54.108.225"
    macaddr: "00:50:56:bd:f7:7a"
#other:
#  - name: "non-cluster-vm"
#    ipaddr: "192.168.7.31"
#    macaddr: "52:54:00:f4:2e:2e"
```
-----------------------------------------------------------------
Corremos el playbook de ansible, el mismo instala bind (dns), dhcp, haproxy (lb) y httpd (webserver), y configura los mismos para la implementacion del cluster.
 ```
[root@helper ~]# ansible-playbook -e @vars.yaml tasks/main.yml
 ```

 ```
[root@helper ~]# cat /etc/named.conf
-----------------------------------------------------------------
//
// named.conf
//
// Provided by Red Hat bind package to configure the ISC BIND named(8) DNS
// server as a caching only nameserver (as a localhost DNS resolver only).
//
// See /usr/share/doc/bind*/sample/ for example named configuration files.
//

options {
        listen-on port 53 { any; };
        listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        allow-query     { any; };

        /*
         - If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
         - If you are building a RECURSIVE (caching) DNS server, you need to enable
           recursion.
         - If your recursive DNS server has a public IP address, you MUST enable access
           control to limit queries to your legitimate users. Failing to do so will
           cause your server to become part of large scale DNS amplification
           attacks. Implementing BCP38 within your network would greatly
           reduce such attack surface
        */
        recursion yes;

        /* Fowarders */
        forward only;
        forwarders { 8.8.8.8; 8.8.4.4; };

        dnssec-enable yes;
        dnssec-validation no;

        managed-keys-directory "/var/named/dynamic";

        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";

        /* https://fedoraproject.org/wiki/Changes/CryptoPolicy */
        /* include "/etc/crypto-policies/back-ends/bind.config"; */
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
        type hint;
        file "named.ca";
};

########### Add what's between these comments ###########
zone "ocp4.lab.local" IN {
        type    master;
        file    "zonefile.db";
};

zone "108.54.10.in-addr.arpa" IN {
        type    master;
        file    "reverse.db";
};
########################################################

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
-----------------------------------------------------------------
  ```
 ```
[root@helper ~]# cat /var/named/zonefile.db
-----------------------------------------------------------------
$TTL 1W
@       IN      SOA     ns1.ocp4.lab.local.     root (
                        2021011200      ; serial
                        3H              ; refresh (3 hours)
                        30M             ; retry (30 minutes)
                        2W              ; expiry (2 weeks)
                        1W )            ; minimum (1 week)
        IN      NS      ns1.ocp4.lab.local.
        IN      MX 10   smtp.ocp4.lab.local.
;
;
ns1     IN      A       10.54.108.220
smtp    IN      A       10.54.108.220
;
helper  IN      A       10.54.108.220
;
;
; The api points to the IP of your load balancer
api             IN      A       10.54.108.220
api-int         IN      A       10.54.108.220
;
; The wildcard also points to the load balancer
*.apps          IN      A       10.54.108.220
;
; Create entry for the local registry
registry        IN      A       10.54.108.220
;
; Create entry for the bootstrap host
bootstrap       IN      A       10.54.108.226
;
; Create entries for the master hosts
master0         IN      A       10.54.108.221
master1         IN      A       10.54.108.222
master2         IN      A       10.54.108.223
;
; Create entries for the worker hosts
worker0         IN      A       10.54.108.224
worker1         IN      A       10.54.108.225
;
; The ETCd cluster lives on the masters...so point these to the IP of the masters
etcd-0  IN      A       10.54.108.221
etcd-1  IN      A       10.54.108.222
etcd-2  IN      A       10.54.108.223
;
; The SRV records are IMPORTANT....make sure you get these right...note the trailing dot at the end...
_etcd-server-ssl._tcp   IN      SRV     0 10 2380 etcd-0.ocp4.lab.local.
_etcd-server-ssl._tcp   IN      SRV     0 10 2380 etcd-1.ocp4.lab.local.
_etcd-server-ssl._tcp   IN      SRV     0 10 2380 etcd-2.ocp4.lab.local.
;
;EOF
-----------------------------------------------------------------
  ```
 ```
[root@helper ~]# cat /var/named/reverse.db
-----------------------------------------------------------------
$TTL 1W
@       IN      SOA     ns1.ocp4.lab.local.     root (
                        2021011200      ; serial
                        3H              ; refresh (3 hours)
                        30M             ; retry (30 minutes)
                        2W              ; expiry (2 weeks)
                        1W )            ; minimum (1 week)
        IN      NS      ns1.ocp4.lab.local.
;
; syntax is "last octet" and the host must have fqdn with trailing dot
1       IN      PTR     helper.ocp4.lab.local.

221     IN      PTR     master0.ocp4.lab.local.
222     IN      PTR     master1.ocp4.lab.local.
223     IN      PTR     master2.ocp4.lab.local.
;
226     IN      PTR     bootstrap.ocp4.lab.local.
;
220     IN      PTR     api.ocp4.lab.local.
220     IN      PTR     api-int.ocp4.lab.local.
;
224     IN      PTR     worker0.ocp4.lab.local.
225     IN      PTR     worker1.ocp4.lab.local.
;
;EOF
-----------------------------------------------------------------
#######################################################################################################################
  ```
 
 ```
[root@helper ~]# cat /etc/dhcp/dhcpd.conf
#######################################################################################################################
authoritative;
ddns-update-style interim;
default-lease-time 14400;
max-lease-time 14400;

    option routers                  10.54.108.1;
    option broadcast-address        10.54.108.255;
    option subnet-mask              255.255.255.0;
    option domain-name-servers      10.54.108.220;
    option domain-name              "ocp4.lab.local";
    option domain-search            "ocp4.lab.local", "lab.local";

    subnet 10.54.108.0 netmask 255.255.255.0 {
    interface ens160;
        pool {
            range 10.54.108.221 10.54.108.229;
        # Static entries
        host bootstrap { hardware ethernet 00:50:56:bd:fd:fc; fixed-address 10.54.108.226; }
        host master0 { hardware ethernet 00:50:56:bd:07:c0; fixed-address 10.54.108.221; }
        host master1 { hardware ethernet 00:50:56:bd:a2:f0; fixed-address 10.54.108.222; }
        host master2 { hardware ethernet 00:50:56:bd:9c:a4; fixed-address 10.54.108.223; }
        host worker0 { hardware ethernet 00:50:56:bd:24:d7; fixed-address 10.54.108.224; }
        host worker1 { hardware ethernet 00:50:56:bd:f7:7a; fixed-address 10.54.108.225; }
        # this will not give out addresses to hosts not listed above
        deny unknown-clients;

        # this is PXE specific
        filename "pxelinux.0";

        next-server 10.54.108.220;
        }
}
#######################################################################################################################
  ```
 ```
[root@helper ~]# cat /etc/haproxy/haproxy.cfg
#######################################################################################################################
#---------------------------------------------------------------------
# Example configuration for a possible web application.  See the
# full configuration options online.
#
#   http://haproxy.1wt.eu/download/1.4/doc/configuration.txt
#
#---------------------------------------------------------------------

#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    #
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    #
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local2

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    tcp
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          4h
    timeout server          4h
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000
 ```

 ```
listen stats
    bind :9000
    mode http
    stats enable
    stats uri /
    monitor-uri /healthz


frontend openshift-api-server
    bind *:6443
    default_backend openshift-api-server
    option tcplog

backend openshift-api-server
    balance source
    server bootstrap 10.54.108.226:6443 check
    server master0 10.54.108.221:6443 check
    server master1 10.54.108.222:6443 check
    server master2 10.54.108.223:6443 check

frontend machine-config-server
    bind *:22623
    default_backend machine-config-server
    option tcplog

backend machine-config-server
    balance source
    server bootstrap 10.54.108.226:22623 check
    server master0 10.54.108.221:22623 check
    server master1 10.54.108.222:22623 check
    server master2 10.54.108.223:22623 check

frontend ingress-http
    bind *:80
    default_backend ingress-http
    option tcplog

backend ingress-http
    balance source
    server worker0-http-router0 10.54.108.224:80 check
    server worker1-http-router1 10.54.108.225:80 check

frontend ingress-https
    bind *:443
    default_backend ingress-https
    option tcplog

backend ingress-https
    balance source
    server worker0-https-router0 10.54.108.224:443 check
    server worker1-https-router1 10.54.108.225:443 check
 ```

Bajamos los paquetes:(openshift-install-linux.tar.gz, openshift-client-linux.tar.gz, rhcos-live.x86_64.iso, pull-secret.txt)
 ```
[root@helper ~]# tar xzvf openshift-install-linux.tar.gz
[root@helper ~]# tar xzvf openshift-client-linux.tar.gz
[root@helper ~]# cp oc kubectl /usr/local/bin 
 ```
Creamos el directorio de instalacion
 ```
[root@helper ~]# mkdir ocp
 ```
Creamos el archivo install-config.yaml
 ```
[root@helper ~]# vim install-config.yaml
###########################################
apiVersion: v1
baseDomain: lab.local
compute:
- hyperthreading: Enabled
  name: worker
  replicas: 0
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: ocp4
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}
fips: false
pullSecret: '{"auths":{"cloud.opens -----------------------"}}}'
sshKey: 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDSGNnoBSKsCeJo7y/jRtKxIx6MQ45xjeGlAFW9Bus06+FywBLaTpfyqcysshBQ738jfui31+aVESiQeATx/UdjDCnD9anNkmmoOqfDd08lmNGA2nP58fgLKP5vmjdwG7NyGIsh2XIRIpklvsu1P+WFPjPyvGUPkAnNsov2j6BMfE4fkxhdRofnKt7d+9sogtIOyCWeE2KEy6ZmQ/bxab10uXvk6/5pmpc0luZG+sj/nk49DjwHThklKe8qbQKSKSfGZzcWYs52wa3NiH5mLo4VV9jYPQ7FhbcwqWwUJwMngLAd/2DU7cKkfGosODg9jquhG13EoPPic9gBcm1VMTQLznWu7S+Aw++h68FNeg9kPXMQD159+S7VrNrPp9jP58v3U229Q1T2PHnoNlmut1WvRU93VLVrb3mcPagQbOI8Y4n7IqYSpb8tg3MuLjDLyB1rZDdL5Yv1MnKjc+R6u/9XapJfuvvuwziTsUeRT4lbl149hnMffjxkTbeU4tSw+qTfjxaGtmtQXHDohnBUm0nbbYxuL30o3NLKDPW8M3ytNPNwxgTYUQlq7lOBLCwiWgPPyvqRzBEeyA/fG2aT7yZYCMLTA+O6R+5ISgDDqMYdmNlAQYQnfv61KU3at5prTHFEvtFwoHC3YnBkBDTpCAmFQzRLMfX+ghzjEy0Cglk+/w== root@helper'
###########################################
 ```
Copiamos el archivo al directorio de instalacion 
 ```
[root@helper ~]# cp install-config.yaml ocp
 ```
Creamos los manifiestos
 ```
[root@helper ~]# ./openshift-install create manifests --dir=ocp
 ```
Creamos los ignitions
 ```
[root@helper ~]# ./openshift-install create ignition-configs --dir=ocp
 ```
Copiamos los ignitions al web server
 ```
[root@helper ~]# cp ocp/*.ign /var/www/html/.
[root@helper ~]# chmod 644 /var/www/html/*.ign
 ```
Creamos las Vms con los minimos requisitos
 ```
Bootstrap (1)
SO: RHCOS
CPUs: 4
MRam: 16 GB
Storage: 120 GB

Master (3)
SO: RHCOS
CPUs: 4
MRam: 16 GB
Storage: 120 GB

Worker (2)
SO: RHCOS or RHEL 7.6
CPUs: 4
MRam: 8 GB
Storage: 120 GB
 ```
Booteamos montando el iso "rhcos-live.x86_64.iso"
Una vez iniciado ejecutar
 ```
# sudo coreos-installer install --ignition-url=http://<helper>:8080/<bootstrap/master/worker>.ign /dev/sda --insecure-ignition
 ```
En nuestro caso
 ```
# sudo coreos-installer install --ignition-url=http://10.54.108.220:8080/bootstrap.ign /dev/sda --insecure-ignition
 ```
 
 Una vez terminada la instalacion esperar a que todos hayan terminado para realizar su reinicio.

Para ver el proceso de instalacion
Nos legueamos al bootstrap 
 ```
[root@helper ~]# ssh -i ~/.ssh/ocp4 core@bootstrap.ocp4.lab.local
 ```
Luego en el bootstrap:
 ```
[core@bootstrap ~]$ journalctl -b -f -u release-image.service -u bootkube.service
 ```
En el helper ejecutar:
 ```
[root@helper ~]# ./openshift-install --dir=ocp wait-for bootstrap-complete --log-level=info
 ```
 ```
# oc get csr -o json --kubeconfig=/home/install/ocp/auth/kubeconfig | jq -r '.items[] | select(.status == {}) | .metadata.name'
csr-7dzvd
csr-xq5vl
 ```
  ```
# oc adm certificate approve csr-7dzvd  --kubeconfig=/home/install/ocp/auth/kubeconfig
certificatesigningrequest.certificates.k8s.io/csr-7dzvd approved
 ```
 ```
# oc adm certificate approve csr-xq5vl  --kubeconfig=/home/install/ocp/auth/kubeconfig
certificatesigningrequest.certificates.k8s.io/csr-xq5vl approved
 ```
  ```
# oc get csr -o json --kubeconfig=/home/install/ocp/auth/kubeconfig  | jq -r '.items[] | select(.status == {}) | .metadata.name'
csr-5j5rz
csr-x4vvr
 ```
  ```
# oc adm certificate approve csr-5j5rz --kubeconfig=/home/install/ocp/auth/kubeconfig
certificatesigningrequest.certificates.k8s.io/csr-5j5rz approved
 ```
  ```
# oc adm certificate approve csr-x4vvr --kubeconfig=/home/install/ocp/auth/kubeconfig
certificatesigningrequest.certificates.k8s.io/csr-x4vvr approved
 ```
 ```
# ./openshift-install wait-for install-complete --dir=ocp  --log-level debug
 ```
La salida final es:
 ```
 minutes per instance)
DEBUG Cluster is initialized
INFO Waiting up to 10m0s for the openshift-console route to be created...
DEBUG Route found in openshift-console namespace: console
DEBUG Route found in openshift-console namespace: downloads
DEBUG OpenShift console route is created
INFO Install complete!
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/home/install/ocp/auth/kubeconfig'
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.ocp4.lab.local
INFO Login to the console with user: "kubeadmin", and password: "CDZCC-CRY6C-TSKbT-hdmJs"
DEBUG Time elapsed per stage:
DEBUG Cluster Operators: 18m44s
INFO Time elapsed: 18m44s
 ```

Luego nos logueamos 
```
# export KUBECONFIG=ocp/auth/kubeconfig
# oc get nodes
NAME                     STATUS   ROLES    AGE    VERSION
master0.ocp4.lab.local   Ready    master   150m   v1.19.0+7070803
master1.ocp4.lab.local   Ready    master   151m   v1.19.0+7070803
master2.ocp4.lab.local   Ready    master   151m   v1.19.0+7070803
worker0.ocp4.lab.local   Ready    worker   107m   v1.19.0+7070803
worker1.ocp4.lab.local   Ready    worker   105m   v1.19.0+7070803
```

### Acceso a la consola web
- url: https://console-openshift-console.apps.ocp4.lab.local
- user: kubeadmin
- pass: CDZCC-CRY6C-TSKbT-hdmJs

### Documentación oficial
[Automatización con ansible](https://github.com/RedHatOfficial/ocp4-vsphere-upi-automation)

[Download isos](https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/)

[Download Install](https://cloud.redhat.com/openshift/install)

[Video de Instalación](https://www.youtube.com/watch?v=d03xg2PKOPg&feature=youtu.be)



