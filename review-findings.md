# Review findings: proposed configuration

These are the changes I would make for the findings in the [README](README.md#security-and-design-review). **They have not been applied to the `.pkt` file or tested**, so treat them as analysis rather than verified configuration.


## 1. DNS: publish the public address
HTTPS & DNS Server → Services → DNS: change `mohammad-ibaad.net` from `192.168.1.3` to **`222.223.50.8`**. Add a second A record `intranet.mohammad-ibaad.net → 192.168.1.3` for inside users, because Packet Tracer doesn't hairpin NAT. Add the record `mail.mohammad-ibaad.net → 222.223.50.9` too.
Check: PC7 → Web Browser → `http://mohammad-ibaad.net` loads.

## 2. BGP: advertise a prefix from AS 100
```
router bgp 100
 network 222.223.50.0 mask 255.255.255.0
```
Caveat: the public NAT IPs sit inside the R3–R4 link subnet, so BGP has nothing *new* to advertise. This fix only makes `show ip bgp` on R4 list a prefix learned from AS 100. Doing it "properly" means moving the public IPs to their own block, which also forces changes to NAT and DNS. That is a larger redesign.
Check: `show ip bgp summary` on R4 shows PfxRcd ≥ 1.

## 3. Restrict AS 200 to the published services (Ex 17)
```
ip access-list extended FROM-AS200
 permit tcp host 222.223.50.4 host 222.223.50.3 eq bgp
 permit tcp host 222.223.50.4 eq bgp host 222.223.50.3
 permit tcp any host 222.223.50.8 eq www
 permit tcp any host 222.223.50.8 eq 443
 permit udp any host 222.223.50.8 eq domain
 permit tcp any host 222.223.50.9 eq smtp
 permit tcp any host 222.223.50.9 eq pop3
 permit icmp any any echo-reply
 deny   ip any any log
interface Serial3/0
 ip access-group FROM-AS200 in
```
Gotchas: the two BGP lines must come first, otherwise the ACL would drop the BGP session itself. The rules match the **public** IPs (.8/.9) because inbound ACLs run before NAT translates the address.
Check: from PC7, the browser and email work, while `ping 222.223.50.8` and FTP to .8 fail. Run `show access-lists` on R3 and check the counters increment.

## 4. Device hardening (every router)
```
hostname R4            ! R4 only
enable secret <strong-password>
service password-encryption
no router rip
ip domain-name mohammad-ibaad.net
username admin secret <strong-password>
crypto key generate rsa      ! answer 1024
ip ssh version 2
line vty 0 4
 login local
 transport input ssh
```
On R1, also add `passive-interface FastEthernet0/0`, `6/0` and `7/0` under `router ospf 1`, so the router doesn't send OSPF hellos to user LANs.
Check: from a PC, `ssh -l admin <router-ip>` works and telnet is refused.

Each fix would end with `copy run start` on the router, followed by its verify step.
