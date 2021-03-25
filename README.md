# IPsec_NAT-T_Example

## Goal
![](./NWD.PNG)
* 左からR1, R2, ..., R5とし、R1とR5のEnd-EndでIPsec Tunnelを確立する。
* R2 == R3 == R4は Internet区間
* R1 == R2 および R4 == R5 はLAN区間
* R2とR4でNAPTを適用。
* NATを経由するIPsecはNAT Traversalとなることに注意する。
  * https://www.infraexpert.com/study/ipsec15.html

## Configs
* R2とR4でのNATにてポートフォワーディング設定を適用する。
* 500と4500両方をフォワーディングすることでIPsec tunnelが確立する。
```
R2: 
ip nat inside source static udp 192.168.0.1 500 1.0.0.2 500 extendable
ip nat inside source static udp 192.168.0.1 4500 1.0.0.2 4500 extendable

R4:
ip nat inside source static udp 192.168.1.5 500 2.0.0.4 500 extendable
ip nat inside source static udp 192.168.1.5 4500 2.0.0.4 4500 extendable
```

## Results
* Cisco ルータの場合、IPsec tunnelが upして確立を確認可能
```
R1#show ip int brie
Interface                  IP-Address      OK? Method Status                Protocol
FastEthernet0/0            unassigned      YES unset  administratively down down
FastEthernet0/1            192.168.0.1     YES manual up                    up  
Tunnel15                   172.24.15.1     YES manual up                    up  

R1#ping 172.24.15.5
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.24.15.5, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 108/122/144 ms
```

* SAの結果でも確認可能（IPsecのログを理解している人向け）
```
R1#show crypto isakmp sa
IPv4 Crypto ISAKMP SA
dst             src             state          conn-id status
192.168.0.1     2.0.0.4         QM_IDLE           1002 ACTIVE

IPv6 Crypto ISAKMP SA

R1#show crypto ipsec sa

interface: Tunnel15
    Crypto map tag: Tunnel15-head-0, local addr 192.168.0.1

   protected vrf: (none)
   local  ident (addr/mask/prot/port): (0.0.0.0/0.0.0.0/0/0)
   remote ident (addr/mask/prot/port): (0.0.0.0/0.0.0.0/0/0)
   current_peer 2.0.0.4 port 4500
     PERMIT, flags={origin_is_acl,}
    #pkts encaps: 5, #pkts encrypt: 5, #pkts digest: 5
    #pkts decaps: 5, #pkts decrypt: 5, #pkts verify: 5
    #pkts compressed: 0, #pkts decompressed: 0
    #pkts not compressed: 0, #pkts compr. failed: 0
    #pkts not decompressed: 0, #pkts decompress failed: 0
    #send errors 0, #recv errors 0

     local crypto endpt.: 192.168.0.1, remote crypto endpt.: 2.0.0.4
     path mtu 1500, ip mtu 1500, ip mtu idb FastEthernet0/1
     current outbound spi: 0x2756FBFC(660012028)
     PFS (Y/N): N, DH group: none

     inbound esp sas:
      spi: 0x4D2DA165(1294836069)
        transform: esp-3des esp-sha-hmac ,
        in use settings ={Tunnel UDP-Encaps, }
        conn id: 11, flow_id: SW:11, sibling_flags 80000040, crypto map: Tunnel15-head-0
        sa timing: remaining key lifetime (k/sec): (4608000/355)
        IV size: 8 bytes
        replay detection support: Y
        Status: ACTIVE(ACTIVE)

     inbound ah sas:

     inbound pcp sas:

     outbound esp sas:
      spi: 0x2756FBFC(660012028)
        transform: esp-3des esp-sha-hmac ,
        in use settings ={Tunnel UDP-Encaps, }
        conn id: 12, flow_id: SW:12, sibling_flags 80000040, crypto map: Tunnel15-head-0
        sa timing: remaining key lifetime (k/sec): (4608000/355)
        IV size: 8 bytes
        replay detection support: Y
        Status: ACTIVE(ACTIVE)

     outbound ah sas:

     outbound pcp sas:
```

* NAT変換もされてることを確認
```
R2#show ip nat translations
Pro Inside global      Inside local       Outside local      Outside global
udp 1.0.0.2:500        192.168.0.1:500    2.0.0.4:500        2.0.0.4:500
udp 1.0.0.2:500        192.168.0.1:500    ---                ---
udp 1.0.0.2:4500       192.168.0.1:4500   2.0.0.4:4500       2.0.0.4:4500
udp 1.0.0.2:4500       192.168.0.1:4500   ---                ---
```

