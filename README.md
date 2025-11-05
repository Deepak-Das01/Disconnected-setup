# Disconnected-setup
Use these details to setup disconnected enviroment

First we are gining to install the required pacakages 
```bash
dnf install bind bind-utils -y
```

[root@bastion dns]# ls
named.conf  zones

[root@bastion dns]# ls -l zones/
total 8
-rw-r--r--. 1 root root 1670 Nov  5 18:43 db.kubelabs.com
-rw-r--r--. 1 root root  824 Nov  5 18:48 db.reverse
[root@bastion dns]#

```bash
[root@bastion dns]# cat named.conf
//
// named.conf
//
// Provided by Red Hat bind package to configure the ISC BIND named(8) DNS
// server as a caching only nameserver (as a localhost DNS resolver only).
//
// See /usr/share/doc/bind*/sample/ for example named configuration files.
//
// See the BIND Administrator's Reference Manual (ARM) for details about the
// configuration located in /usr/share/doc/bind-{version}/Bv9ARM.html

options {
        listen-on port 53 { 127.0.0.1; 10.9.8.1; };
#       listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        recursing-file  "/var/named/data/named.recursing";
        secroots-file   "/var/named/data/named.secroots";
        allow-query     { localhost; 10.9.8.0/24; };

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

        dnssec-enable yes;
        dnssec-validation yes;

        # Using Google DNS
        forwarders {
                8.8.8.8;
                8.8.4.4;
        };

        /* Path to ISC DLV key */
        bindkeys-file "/etc/named.root.key";

        managed-keys-directory "/var/named/dynamic";

        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";
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

# Include ocp zones

zone "kubelabs.com" {
    type master;
    file "/etc/named/zones/db.kubelabs.com"; # zone file path
};

zone "8.9.10.in-addr.arpa" {
    type master;
    file "/etc/named/zones/db.reverse";  # 192.168.22.0/24 subnet
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
[root@bastion dns]#
```

```bash
[root@bastion dns]# cat zones/db.kubelabs.com
$TTL    604800
@       IN      SOA     bastion.kubelabs.com. contact.kubelabs.com (
                  1     ; Serial
             604800     ; Refresh
              86400     ; Retry
            2419200     ; Expire
             604800     ; Minimum
)
        IN      NS      bastion

bastion.kubelabs.com.          IN      A       10.9.8.1

; Temp Bootstrap Node
ocp-bootstrap.kubelabs.com.        IN      A      10.9.8.10

; Control Plane Nodes
master01.kubelabs.com.         IN      A      10.9.8.11
master02.kubelabs.com.         IN      A      10.9.8.12
master03.kubelabs.com.         IN      A      10.9.8.13

; Worker Nodes
worker01.kubelabs.com.        IN      A      10.9.8.21
worker02.kubelabs.com.        IN      A      10.9.8.22

; Edge Nodes
edge01.kubelabs.com.        IN      A      10.9.8.31
edge02.kubelabs.com.        IN      A      10.9.8.32

; OpenShift Internal - Load balancer
api.kubelabs.com.        IN    A    10.9.8.1
api-int.kubelabs.com.    IN    A    10.9.8.1
*.apps.kubelabs.com.     IN    A    10.9.8.1

; ETCD Cluster
etcd-0.kubelabs.com.    IN    A     10.9.8.15
etcd-1.kubelabs.com.    IN    A     10.9.8.16
etcd-2.kubelabs.com.    IN    A     10.9.8.17

; OpenShift Internal SRV records (cluster name = lab)
_etcd-server-ssl._tcp.kubelabs.com.    86400     IN    SRV     0    10    2380    etcd-0.kubelabs
_etcd-server-ssl._tcp.kubelabs.com.    86400     IN    SRV     0    10    2380    etcd-1.kubelabs
_etcd-server-ssl._tcp.kubelabs.com.    86400     IN    SRV     0    10    2380    etcd-2.kubelabs

oauth-openshift.apps.kubelabs.com.     IN     A     10.9.8.1
console-openshift-console.apps.kubelabs.com.     IN     A     10.9.8.1
[root@bastion dns]#
```

```bash
[root@bastion dns]# cat zones/db.reverse
$TTL 604800
@   IN  SOA  bastion.kubelabs.com. contact.kubelabs.com (
        2025110501 ; Serial (use date-based format)
        604800     ; Refresh
        86400      ; Retry
        2419200    ; Expire
        604800 )   ; Minimum TTL

    IN  NS  bastion.kubelabs.com.

; --- PTR Records (Reverse DNS) ---

1    IN  PTR  bastion.kubelabs.com.
1    IN  PTR  api.kubelabs.com.
1    IN  PTR  api-int.kubelabs.com.

10   IN  PTR  ocp-bootstrap.kubelabs.com.

11   IN  PTR  master01.kubelabs.com.
12   IN  PTR  master02.kubelabs.com.
13   IN  PTR  master03.kubelabs.com.

21   IN  PTR  worker01.kubelabs.com.
22   IN  PTR  worker02.kubelabs.com.

31   IN  PTR  edge01.kubelabs.com.
32   IN  PTR  edge02.kubelabs.com.

15   IN  PTR  etcd-0.kubelabs.com.
16   IN  PTR  etcd-1.kubelabs.com.
17   IN  PTR  etcd-2.kubelabs.com.

```

[root@bastion dns]#
\cp ./ocp4-metal-install/dns/named.conf /etc/named.conf
cp -R ./ocp4-metal-install/dns/zones /etc/named/

systemctl enable named
systemctl start named
systemctl status named



