# Introduction

This document describes the configuration steps necessary to secure a BGP session with TCP-AO between Arista EOS routers and Juniper routers.

There are two small sections of configuration that are needed prior to using TCP-AO on a BGP neighbor:
1. The key-chain, which includes the secret being used, key lifetime, and crypto algorithm
1. A TCP-AO section, which references the above key-chain and adds  AO specific parameters, such as send-ID and receive-ID

This topology of this test:
```

     ┌────────────────┐                 ┌────────────────┐
     │                │      BGP        │                │
     │  Arista vEOS   │  AO Key ID 10   │  Arista vEOS   │
     │    4.29.2F     ├─────────────────┤    4.29.2F     │
     │                │  hmac-sha1-96   │                │
     └───────┬────────┘                 └───────┬────────┘
             │                                  │
             │                                  │
        BGP  │ AO Key ID 0         AO Key ID 10 │  BGP
             │ hmac-sha1-96     aes-128-cmac-96 │
             │                                  │
             │        ┌────────────────┐        │
             │        │                │        │
             │        │  Juniper vMX   │        │
             └────────┤  21.2R3-S2.9   ├────────┘
                      │                │
                      └────────────────┘
```  

TCP-AO was added to Arista EOS 2.48.2F.</br>
This document describes the configuration steps necessary to secure a BGP session with TCP-AO between an Arista device running EOS 4.29.1F and a Juniper cRPD running 22.4R1.10.

## Prerequisites 
* Arista EOS 4.28.2F or higher
* Multi-agent routing model has to be configured
* TCP-AO is available on all platforms

Documentation on this feature can be found on the Arista website as a TOI:</br>
https://www.arista.com/en/support/toi/eos-4-28-2f/16087-bgp-tcp-authentication-option-tcp-ao

### EOS configuration
#### Enable multi-agent model for routing daemons
```
service routing protocols model multi-agent
```
Please note that changing the routing model from `ribd` to `multi-agent` requires a reload on the system.

#### Shared Secret configuraiton
##### Configuration of the Shared Secret profile
```
management security
   session shared-secret profile BGP
      secret 10 7 $1c$zXHy2/5IOz6JEc5qRNYMBA== receive-lifetime 2023-01-01 00:00:00 infinite transmit-lifetime 2023-01-01 00:00:00 infinite
      secret 0 7 $1c$zXHy2/5IOz6JEc5qRNYMBA== receive-lifetime 2023-01-01 00:00:00 infinite transmit-lifetime 2023-01-01 00:00:00 infinite
```

Multiple secrets with different lifetimes can be configured as well to ensure proper key rollover.

##### Verification of the Shared Secret profile
```
arista#show management security session shared-secret profile BGP
Profile: BGP

Current receive secret: ID: 10, Expires: Does not expire
Current transmit secret: ID: 10, Expires: Does not expire

Receive secret rotation order: 10
Transmit secret rotation order: 10

Secrets:
   ID 10
      Secret: $1c$zXHy2/5IOz6JEc5qRNYMBA==
      Receive lifetime: January 01 2023, 00:00 UTC to infinite
      Transmit lifetime: January 01 2023, 00:00 UTC to infinite
   ID 0
      Secret: $1c$zXHy2/5IOz6JEc5qRNYMBA==
      Receive lifetime: January 01 2023, 00:00 UTC to infinite
      Transmit lifetime: January 01 2023, 00:00 UTC to infinite
```

#### BGP configuration
##### Configuration of the BGP neighbor

On `veos1`:
```
router bgp 65001
   neighbor default send-community
   neighbor 10.255.255.2 remote-as 65002
   neighbor 10.255.255.2 password shared-secret profile BGP algorithm hmac-sha1-96
   neighbor 10.255.255.6 remote-as 65006
   neighbor 10.255.255.6 password shared-secret profile BGP algorithm hmac-sha1-96
```

On `veos2`:
```
router bgp 65006
   neighbor default send-community
   neighbor 10.255.255.5 remote-as 65001
   neighbor 10.255.255.5 password shared-secret profile BGP algorithm hmac-sha1-96
   neighbor 10.255.255.10 remote-as 65002
   neighbor 10.255.255.10 password shared-secret profile BGP algorithm aes-128-cmac-96
```

### Junos configuration
#### Configuration of the Authentication Key Chain
```
authentication-key-chains {
    key-chain AES {
        key 10 {
            secret "$9$kP5FCA0O1hqm6ApBSy"; ## SECRET-DATA
            start-time "2023-1-1.00:00:00 +0000";
            algorithm ao;
            ao-attribute {
                send-id 10;
                recv-id 10;
                tcp-ao-option enabled;
                cryptographic-algorithm aes-128-cmac-96;
            }
        }
    }
    key-chain SHA {
        key 55 {
            secret "$9$Dyk.5n6Atu1jHz69pRE"; ## SECRET-DATA
            start-time "2023-1-1.00:00:00 +0000";
            algorithm ao;
            ao-attribute {
                send-id 0;
                recv-id 0;
                tcp-ao-option enabled;
                cryptographic-algorithm hmac-sha-1-96;
            }
        }
    }
}
```

#### Configuration of the BGP neighbor
```
group VEOS1 {
    authentication-algorithm ao;
    authentication-key-chain SHA;
    peer-as 65001;
    neighbor 10.255.255.1;
}
group VEOS2 {
    authentication-algorithm ao;
    authentication-key-chain AES;
    peer-as 65006;
    neighbor 10.255.255.9;
}
```

## Results
### Verification on Arista EOS

On `veos1`:
```
veos1#show bgp neighbors | sec TCP-AO
TCP-AO Authentication:
  Profile: BGP
  MAC algorithm: hmac-sha1-96
  Current key ID: 0
  Next receive key ID: 0
  Active receive key IDs: 0, 10
TCP-AO Authentication:
  Profile: BGP
  MAC algorithm: hmac-sha1-96
  Current key ID: 10
  Next receive key ID: 10
  Active receive key IDs: 0, 10
  ```

On `veos2`:
```
veos2#show bgp neighbors | sec TCP-AO
TCP-AO Authentication:
  Profile: BGP
  MAC algorithm: hmac-sha1-96
  Current key ID: 10
  Next receive key ID: 10
  Active receive key IDs: 10
TCP-AO Authentication:
  Profile: BGP
  MAC algorithm: aes-128-cmac-96
  Current key ID: 10
  Next receive key ID: 10
  Active receive key IDs: 10
```
  ### Verification on Juniper JunOS 
  ```
admin@vmx1> show security keychain detail 
Keychain                 Active-ID        Next-ID       Transition  Tolerance
                       Send   Receive   Send   Receive
 AES                     10     10       None   None     None       3600 (secs)
  Id 10, Algorithm hmac-md5, State send-receive, Option basic
  Start-time Sun Jan  1 00:00:00 2023, Mode send-receive
Keychain                 Active-ID        Next-ID       Transition  Tolerance
                       Send   Receive   Send   Receive
 SHA                     55     55       None   None     None       3600 (secs)
  Id 55, Algorithm hmac-md5, State send-receive, Option basic
  Start-time Sun Jan  1 00:00:00 2023, Mode send-receive
  
 ```
 ```
  admin@vmx1> show bgp neighbor | match Authentication 
  Authentication key chain: SHA
  Authentication algorithm: ao
  Authentication key chain: AES
  Authentication algorithm: ao 
  ```
