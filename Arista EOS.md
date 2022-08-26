# Introduction

TCP-AO was added to Arista EOS 2.48.2F..  This document describes the configuration steps necessary to secure a BGP session with TCP-AO between an Arista switch running EOS 4.28.2F and a Juniper MX running 20.4R3.8.

## Prerequisites 
Feature documentation is a bit sparse.  Below is a very basic configuration using default crypto algorithms (hmac-sha1-96) and infinite key lifetimes.  Key rolling was not tested.

In order to use TCP-AO, you must be running the multi-agent model for routing protocols by issuing the configuration command `service routing protocols model multi-agent`


### EOS configuration
#### enable multi-agent model for routing daemons
```
service routing protocols model multi-agent
```

#### define the shared-secret profile
```
management security
   !
   session shared-secret profile profile_TCPAO
      secret 0 7 $1c$YdI3N1Cp7qM+MU/OHPKqemlWZYiipG9b receive-lifetime infinite transmit-lifetime infinite
```
### apply to a BGP neighbor
```
router bgp 65111
   router-id 2.2.2.2
   neighbor 172.20.7.200 remote-as 4200011039
   neighbor 172.20.7.200 password shared-secret profile profile_TCPAO algorithm hmac-sha1-96

```

### Junos configuration
#### define the key-chain
```
security {
    authentication-key-chains {
        key-chain lab-AK7 {
            key 0 {
                secret "$9$CayRtpOIEylK8RhwgoaiHPfT39pB1hrK8GDtOREeKVwY"; ## SECRET-DATA
                start-time "2022-8-26.08:38:16 -0400";
                algorithm ao;
                ao-attribute {
                    send-id 0;
                    recv-id 0;
                    tcp-ao-option enabled;
                }
            }
        }

```

#### apply to a BGP peer
```
protocols {
    bgp {
        group labA7K {
            authentication-algorithm ao;
            authentication-key-chain lab-AK7;
            peer-as 65111;
            neighbor 172.20.7.100;
        }
    }
}
```

## Results
### EOS reports TCP-AO information in the BGP neighbor output
```
show bgp neighbors | begin TCP-AO
TCP-AO Authentication:
  Profile: profile_TCPAO
  MAC algorithm: hmac-sha1-96
  Current key ID: 0
  Next receive key ID: 0
  Active receive key IDs: 0
  ```

  ### Junos 
  ```
  show security keychain detail
Keychain                 Active-ID        Next-ID       Transition  Tolerance
                       Send   Receive   Send   Receive
 lab-AK7                 0      0        None   None     None       3600 (secs)
  Id 0, Algorithm hmac-md5, State send-receive, Option basic
  Start-time Fri Aug 26 08:38:16 2022, Mode send-receive
  ```
