# Introduction

This document describes the configuration steps necessary to secure a BGP session with TCP-AO between two Cisco IOS-XR routers running Version 7.0.2.

There are two small sections of configuration that are needed prior to using TCP-AO on a BGP neighbor:
1. The key-chain, which includes the secret being used, key lifetime, and crypto algorithm
1. A TCP-AO section, which references the above key-chain and adds  AO specific parameters, such as send-ID and receive-ID

This topology of this test:
```

     ┌───────────────┐
     │    IOS XR-a   │─────────────────────┐
     └───────────────┘             ┌───────┴───────┐
            |                      │   Junos OS    │
     ┌───────────────┐             └───────┬───────┘
     │    IOS XR-b   │─────────────────────┘
     └───────────────┘

```  
  
&nbsp;

### Configuration for XR-a
```buildoutcfg
key chain kcTCO-AO-1
 key 0
  accept-lifetime 00:00:00 june 28 2021 infinite
  key-string password 
  send-lifetime 00:00:00 june 28 2021 infinite
  cryptographic-algorithm HMAC-SHA1-96
 !
!
```
*note*: `key 0` does not refer to the send-ID or recv-ID.  This is a locally significant identifier.

```buildoutcfg
tcp ao
 keychain kcTCO-AO-1
  key 0 SendID 0 ReceiveID 0
 !
!
```

The key is used in a BGP peer by:
```buildoutcfg
router bgp 64501
 bgp router-id 172.16.12.1
 address-family ipv4 unicast
  redistribute static
 !
 neighbor 172.16.12.2
  remote-as 64502
  ao kcTCO-AO-1 include-tcp-options enable
  address-family ipv4 unicast
  !
 !
!
```

To verify:
```buildoutcfg
RP/0/RP0/CPU0:XR702-a#show tcp authentication keychain all
Tue Jun 29 22:05:34.922 EDT

Keychain name: kcTCO-AO-1, configured for tcp-ao
Desired key: 0
Total number of keys: 1
Key details:
    Key ID: 0, Active, Valid
Total number of usable (Active & Valid) keys: 1
    Keys: 0, 
```


# Configuration for interop testing between Junos 20.4R2.7 and XR 7.0.2

IOS XR-a
```
tcp ao
 keychain MXinterop
  key 0 SendID 0 ReceiveID 0
 !
 
key chain MXinterop
 key 0
  accept-lifetime 00:00:00 june 25 2021 infinite
  key-string password 011A08105E19091F
  send-lifetime 00:00:00 june 25 2021 infinite
  cryptographic-algorithm HMAC-SHA1-96
 !
 
 
router bgp 64501
 bgp router-id 172.16.12.1
 address-family ipv4 unicast
  redistribute static

 neighbor 172.16.136.101
  remote-as 64901
  ao MXinterop include-tcp-options enable
  ebgp-multihop 8
  address-family ipv4 unicast
  !
 !
!
```

IOS-XR-b
```
tcp ao
 keychain MXinterop
  key 0 SendID 0 ReceiveID 0
 !

key chain MXinterop
 key 0
  accept-lifetime 00:00:00 june 25 2021 infinite
  key-string password 011A08105E19091F
  send-lifetime 00:00:00 june 25 2021 infinite
  cryptographic-algorithm HMAC-SHA1-96
 !
!


router bgp 64502
 bgp router-id 172.16.12.2
 address-family ipv4 unicast
  redistribute static

 neighbor 172.16.136.101
  remote-as 64901
  ao MXinterop include-tcp-options enable
  ebgp-multihop 8
  address-family ipv4 unicast
  !
 !
!
```

Juniper MX
```
 show configuration security authentication-key-chains key-chain XR-interop
key 0 {
    secret "$9$.PT3REyv87CtMX-waJz36"; ## SECRET-DATA
    start-time "2021-6-29.00:00:00 +0000";
    algorithm ao;
    ao-attribute {
        send-id 0;
        recv-id 0;
        tcp-ao-option enabled;
    }
}


 show configuration protocols bgp group XRlab
type external;
multihop {
    ttl 8;
}
import po_REJECT-ALL;
authentication-algorithm ao;
authentication-key-chain XR-interop;
export po_REJECT-ALL;
neighbor 172.16.12.170 {
    peer-as 64501;
}
neighbor 172.16.12.192 {
    peer-as 64502;
}
```

XR show commands (from router `a`)
```
RP/0/RP0/CPU0:XR702-a#show bgp sum
Wed Jun 30 14:29:29.282 EDT
BGP router identifier 172.16.12.1, local AS number 64501
<output truncated>

Neighbor        Spk    AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down  St/PfxRcd
172.16.136.101   0  64901      99      87        5    0    0 00:34:01          0!


RP/0/RP0/CPU0:XR702-a#show tcp authentication keychain MXinterop detail
Wed Jun 30 14:31:26.946 EDT

Keychain name: MXinterop, configured for tcp-ao
Desired key: 0
Detail of last notification from keychain:
Time: 'Jun 30 13:55:21.447.307', event: Config update, attr: Crypto algorithm, key: 0
Total number of keys: 1
Key details:
    Key ID: 0, Active, Valid
    Active_state: 1, invalid_bits: 0x0, state: 0x3
    Key is configured for tcp-ao, Send ID: 0, Receive ID: 0
    Crypto algorithm: HMAC_SHA1_96, key string chksum: 0000373d
    Detail of last notification from keychain:
    Time: 'Jun 30 13:55:21.447.308', event: Config update, attr: Crypto algorithm
    No valid overlapping key
    No keys invalidated

Total number of usable (Active & Valid) keys: 1
    Keys: 0,

Total number of Send IDs: 1
Send ID details:
    SendID: 0, Total number of keys: 1
        Keys: 0,

Total number of Receive IDs: 1
Receive ID details:
    ReceiveID: 0, Total number of keys: 1
        Keys: 0,

```
Juniper MX  show commands
```
 show bgp summary
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 2 Peers: 3 Down peers: 1
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
inet.0
                       0          0          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
172.16.12.170        64501         76         84       0       0       37:07 Establ
  inet.0: 0/0/0/0
172.16.12.192        64502         75         82       0       0       36:36 Establ
  inet.0: 0/0/0/0


show security keychain detail
Keychain                 Active-ID        Next-ID       Transition  Tolerance
                       Send   Receive   Send   Receive
 XR-interop              0      0        None   None     None       3600 (secs)
  Id 0, Algorithm hmac-md5, State send-receive, Option basic
  Start-time Tue Jun 29 00:00:00 2021, Mode send-receive


```

