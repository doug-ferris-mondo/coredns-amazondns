# coredns-amazondns
The *amazondns* plugin behaves **Authoritative name server** using [Amazon DNS Server](https://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_DHCP_Options.html#AmazonDNS) as the backend.

The Amazon DNS server is used to resolve the DNS domain names that you specify in a private hosted zone in Route 53. However, the server acts as **Caching name server**. Although CoreDNS has [proxy plugin](https://github.com/coredns/coredns/tree/master/plugin/proxy) and we can configure Amazon DNS server as the backend, it can't be Authoritative name server. In my case, Authoritative name server is required to handle delegated responsibility for the subdomain. That's why I created this plugin. 

## Name

*amazondns* - enables serving Authoritative name server using Amazon DNS Server as the backend.

## Syntax

```txt
amazondns ZONE [ADDRESS] {
    soa RR
    ns RR
    nsa RR
}
```

* **ZONE** the zone scope for this plugin.
* **ADDRESS** defines the Amazon DNS server address specifically.
  If no **ADDRESS** entry, this plugin resovles it automatically using [Instance Metadata](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html).
* **soa** **RR** SOA record with [RFC 1035](https://tools.ietf.org/html/rfc1035#section-5) style.
* **ns** **RR** NS record(s) with [RFC 1035](https://tools.ietf.org/html/rfc1035#section-5) style.
* **nsa** **RR** A record(s) for the NS(s) with [RFC 1035](https://tools.ietf.org/html/rfc1035#section-5) style.

## Examples

Create your Route 53 private hostead zone with `sub.example.org` and attach your VPC. Then add A record for `test.sub.example.org` into the zone.

Next, boot EC2 instance and deploy CoreDNS binary, and configure CoreDNS config file as below.

```txt
. {
    amazondns sub.example.org {
        soa "sub.example.org 60 IN SOA ns1.sub.example.org hostmaster.sub.example.org (1 7200 900 1209600 86400)"
        ns "sub.example.org 60 IN NS ns1.sub.example.org"
        ns "sub.example.org 60 IN NS ns2.sub.example.org"
        nsa "ns1.sub.example.org 60 IN A 192.168.0.1"
        nsa "ns2.sub.example.org 60 IN A 192.168.0.2"
    }
}
```

Boot CoreDNS and check it how it works.

The `test.sub.example.org` is resolved with *AUTHORITY SECTION* and *ADDITIONAL SECTION* as below.

```bash
> dig @localhost test.sub.example.org +norecurse

; <<>> DiG 9.11.1 <<>> @localhost test.sub.example.org +norecurse
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 28681
;; flags: qr aa ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: 23246de45b4a3601 (echoed)
;; QUESTION SECTION:
;test.sub.example.org.		IN	A

;; ANSWER SECTION:
test.sub.example.org.	60	IN	A	10.0.0.10

;; AUTHORITY SECTION:
sub.example.org.	60	IN	NS	ns1.sub.example.org.
sub.example.org.	60	IN	NS	ns2.sub.example.org.

;; ADDITIONAL SECTION:
ns1.sub.example.org.    60  IN  A   192.168.0.1
ns2.sub.example.org.    60  IN  A   192.168.0.2

;; Query time: 12 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Tue Feb 20 15:11:55 JST 2018
;; MSG SIZE  rcvd: 146
```

Also it can return NS record(s) for subdomain as below.

```bash
> dig @localhost sub.example.org ns

; <<>> DiG 9.11.1 <<>> @localhost sub.example.org ns +norecurse
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 2719
;; flags: qr aa ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: c1c3332966dba8fd (echoed)
;; QUESTION SECTION:
;sub.example.org.      IN  NS

;; ANSWER SECTION:
sub.example.org.   60  IN  NS  ns1.sub.example.org.
sub.example.org.   60  IN  NS  ns2.sub.example.org.

;; ADDITIONAL SECTION:
ns1.sub.example.org.   60  IN  A   192.168.0.1
ns2.sub.example.org.   60  IN  A   192.168.0.2

;; Query time: 1 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Tue Feb 20 15:08:27 JST 2018
;; MSG SIZE  rcvd: 125
```

