--- 
title: BIND9 setup for local homelab
date: 2024-03-05
categories: homelab 
tags: DNS, BIND
---

For my homelab, I will be using BIND9 as my authoriatative DNS server. We will use DNS
failover in a Master/Slave configuraton and configure dynamic updates.

Both servers will be VMs on my proxmox node running CentOS Stream 9.

# Master Server

## Install, Enable Firewall

Install packages and enable firewall. My firewall is firewalld. DNS uses
tcp/udp port 53. The hostname for this machine is dns1.

```bash
[dns1] # dnf install -y bind bind-utils
[dns1] # firewall-cmd --zone=public --add-service=udp --permanent
[dns1] # systemctl enable --now named
```
## Log configuration

For recommended logging configuration, see the offical BIND9 documentation
[here](https://kb.isc.org/docs/aa-01526). Create the directory and give proper
permissions to named service

```bash
[dns1] # mkdir /var/log/named
[dns1] # chmod 0700 /var/log/named
[dns1] # chown named:named /var/log/named
```

## Master /etc/named.conf

BIND9 is configured with /etc/named.conf. Since this is an authoriatative name
server only, I will disable recursion and only enable queries from local
clients.

```bash
include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
include "/etc/rndc.key";

// Allow rndc management

controls {
	inet 127.0.0.1 port 953 allow { 127.0.0.1; } keys { "rndc-key"; };
};

// Limit access to LAN

acl "clients" {
    127.0.0.0/8;
    10.10.110.0/24;
};

options {
    listen-on port 53 { 127.0.0.1; 10.10.110.21; }; 
    listen-on-v6 { none; };
    directory 	"/var/named";
    dump-file 	"/var/named/data/cache_dump.db";
    statistics-file "/var/named/data/named_stats.txt";
    memstatistics-file "/var/named/data/named_mem_stats.txt";

    version none;
    hostname none;
    server-id none;

    recursion no; 

    allow-query { clients; };
    allow-transfer { localhost; 10.11.110.22; }; 

    auth-nxdomain no;
    notify no;
    dnssec-validation auto;

    bindkeys-file "/etc/named.iscdlv.key";
    managed-keys-directory "/var/named/dynamic";
    pid-file "/run/named/named.pid";
    session-keyfile "/run/named/session.key";
};

logging {

    channel "default_log" {
        file "/var/log/named/default.log" versions 10 size 5m;
        print-time yes;
        print-category yes;
        print-severity yes;
        severity info;
    };

    category default { default_log; };
    category general { default_log; };
    category client { default_log; };
    category security { default_log; };
    category lame-servers { null; };
    category queries { default_log; };
    category query-errors { default_log; };
};

zone "home.arpa" {
    type master;
    file "master/master.home.arpa";
    allow-update { key rndc-key; };
    notify yes;
};

zone "110.10.10.in-addr.arpa" {
    type master;
    file "10.10.110.rev";
    allow-update { key rndc-key; };
    notify yes;
};
```

## Master zone file

A simple master zone file is below. 

```bash
$ORIGIN home.arpa. 
$TTL 86400         ; 1 day
home.arpa.
                                2024010100 ; serial
                                3600       ; refresh (1 hour)
                                3600       ; retry (1 hour)
                                604800     ; expire (1 week)
                                3600       ; minimum (1 hour)
                                )
                        NS      dns1.home.arpa.
                        NS      dns2.home.arpa.
                        A       10.10.110.21
                        A       10.10.110.22
$ORIGIN home.arpa.
dns1                    A       10.10.110.21
dns2                    A       10.10.110.22
pve                     A       10.10.110.10
host1                   A       10.10.110.50
```
## Validation

BIND9 includes two helpful utilites to validate configuration:
*named-checkconf* and *named-checkzone*. If no errors are found,
*named-checkconf* will return 0 or no output. For *named-checkzone*, provide
the zone and zone file like so:

```bash
[dns1] # named-checkzone home.arpa /var/named/master/master.home.arpa
zone home.arpa/IN: loaded serial 2024010100
OK
```

## Permissions and SELinux

SELinux is a LSM or Linux Security Module. There are multiple LSMs provided by
the Linux Kernel. Since my distro of choice is CentOS, I will use SELinux as
it's already installed.

The purpose of LSM is MAC or mandatory access control. This basically provides
strict control over access to kernel functions and objects. SELinux and LSMs in
general are complex to say the least so this post will only
provide some basic commands and configurations related to the BIND9 server. 
For more info, see sources at end of this post.

After creating your zone files and /etc/named.conf, we will need to fix
ownership and SELinux file type.

```bash
[dns1] # chown -R named:named /var/named/master
[dns1] # chown named:named /var/named/10.10.110.rev
[dns1] # semanage fcontext -a -t named_zone_t /var/named/master
[dns1] # restorecon -Rv /var/named/
```

Files inherit the policies of their parent directory. In this example, the *.rev*
file was created in /var/named so it has the right context but wrong ownership.
The master file is my custom location so I need to set the new
context. 

To allow named to write to master zone files, verify that
the correct SELinux switch is enabled:

```bash
[dns1] # getsebool -a | grep named
named_tcp_bind_http_port --> off
named_write_master_zones --> on
```
# Slave configuration

## Slave /etc/named.conf

The distinction for a slave /etc/named.conf file is the zone files have a
*type slave* statement and masters are defined.

```bash
include "/etc/named.rfc1912.zones";
include "/etc/named.root.key"

acl "clients" {
    127.0.0.0/8;
    10.10.110.0/24;
};

options {
    listen-on port 53 { 127.0.0.1; 10.10.110.22; }; 
    listen-on-v6 { none; };
    directory 	"/var/named";
    dump-file 	"/var/named/data/cache_dump.db";
    statistics-file "/var/named/data/named_stats.txt";
    memstatistics-file "/var/named/data/named_mem_stats.txt";

    version none;
    hostname none;
    server-id none;

    recursion no; 

    allow-query { clients; };
    allow-transfer { localhost; 10.11.110.22; }; 

    auth-nxdomain no;
    notify no;
    dnssec-validation auto;

    bindkeys-file "/etc/named.iscdlv.key";
    managed-keys-directory "/var/named/dynamic";
    pid-file "/run/named/named.pid";
    session-keyfile "/run/named/session.key";
};

logging {

		channel "default_log" {
				file "/var/log/named/default.log" versions 10 size 5m;
				print-time yes;
				print-category yes;
				print-severity yes;
				severity info;
		};

		category default { default_log; };
		category general { default_log; };
		category client { default_log; };
		category security { default_log; };
		category lame-servers { null; };
		category queries { default_log; };
		category query-errors { default_log; };
};

zone "home.arpa" in{
    type slave;
    file "slaves/slave.home.arpa";
    masters { 10.10.110.21; };
    allow-notify { 10.10.110.21; };
};

zone "110.10.10.in-addr.arpa" {
    type slave;
    file "10.10.110.rev";
    masters { 10.10.110.21; };
    allow-notify { 10.10.110.21; };
};
```

## rndc

rndc is a command-line utility that allows local adminstration for the named
service. This tool allows dynamic updates. In /etc/named.conf, the **controls**
statement specifies the key to use. 

```bash
controls {
    inet 127.0.0.1 port 953 allow { 127.0.0.1; } keys { "rndc-key"; };
};
```

This block allows local adminstration only on port 953 with the generated
rndc-key

```bash
[dns1] rndc-confgen -a -b 512
[dns1] chown root:named /etc/rndc.key
[dns1] chmod 0640 /etc/rndc.key
```
To check the status run *rndc status* :

```bash
[dns1] rndc status
version: BIND 9.16.23-RH (Extended Support Version)
<id:fde3b1f> (version.bind/txt/ch disabled)
running on dns1: Linux x86_64 5.14.0-427.el9.x86_64 #1 SMP PREEMPT_DYNAMIC Fri
Feb 23 04:45:07 UTC 2024
boot time: Wed, 06 Mar 2024 02:44:57 GMT
last configured: Wed, 06 Mar 2024 03:08:23 GMT
configuration file: /etc/named.conf
CPUs found: 2
worker threads: 2
UDP listeners per interface: 2
number of zones: 12 (0 automatic)
debug level: 0
xfers running: 0
xfers deferred: 0
soa queries in progress: 0
query logging is ON
recursive clients: 0/900/1000
tcp clients: 0/150
TCP high-water: 2
server is up and running
```

To update a dynamic zone, use the following workflow:

```bash
$ sudo nsupdate 
> update add test.home.arpa 86400 A 10.10.110.40
> send
> quit
```

If you prefer to manually edit the zone file, first freeze the zone, then edit
and then thaw:

```bash
$ sudo rndc freeze home.arpa
$ sudo rndc reload home.arpa
$ sudo rndc thaw home.arpa
```


# Sources

- [BIND9 Admin Reference](https://bind9.readthedocs.io/en/latest/index.html)
- [BIND9 and
SELinux](https://www.isc.org/docs/2021-09-webinar-BIND-9-Security-SELinux.pdf)
- [RHEL9 BIND9
Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/managing_networking_infrastructure_services/assembly_setting-up-and-configuring-a-bind-dns-server_networking-infrastructure-services)

