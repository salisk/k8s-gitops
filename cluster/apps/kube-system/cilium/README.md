# Cilium

## UniFi BGP

```sh
router bgp 64321
  bgp router-id 192.168.30.1
  no bgp ebgp-requires-policy

  neighbor k8s peer-group
  neighbor k8s remote-as 64322

  neighbor 192.168.30.132 peer-group k8s
  neighbor 192.168.30.133 peer-group k8s
  neighbor 192.168.30.134 peer-group k8s

  address-family ipv4 unicast
    neighbor k8s next-hop-self
    neighbor k8s soft-reconfiguration inbound
  exit-address-family
exit
```
