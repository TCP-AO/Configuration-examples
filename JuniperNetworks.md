# Introduction

This document describes the basic steps needed in order to configure a BGP session using TCP-AO to secure the session.  

Enabling TCP-AO is quite simple and has two steps: 
1. Configure the key
2. Configure the BGP neighbor

Keychain Configuration for TCP AO ( Master Key Tuples ):
----

```
# show security authentication-key-chains
key-chain ao_aes_chain {
    key 0 {
        secret "$9$xk3NVYq.53/taZnCu1yrwYg4UHf5F/A0z3"; ## SECRET-DATA
        start-time "2020-6-16.01:00:00 +0530";
        algorithm ao;
        ao-attribute {
            send-id 9;
            recv-id 2;
            tcp-ao-option enabled;
            cryptographic-algorithm aes-128-cmac-96;
        }
    }
}
key-chain ao_hmac_chain {
    key 0 {
        secret "$9$X9W7dskqfF6AoJ39pBSybs2gGiPfz6CuQF"; ## SECRET-DATA
        start-time "2020-6-16.01:00:00 +0530";
        algorithm ao;
        ao-attribute {
            send-id 9;
            recv-id 2;
            tcp-ao-option enabled;
            cryptographic-algorithm hmac-sha-1-96;
        }
    }
}
```

BGP Configuration ( IPV4 and IPV6 ) for TCP AO association:
----
```
# show protocols
bgp {
    group ebgp_nokia {
        type external;
        local-address 116.197.187.117;
        peer-as 38016;
        local-as 10458;
        neighbor 124.252.255.66 {
            authentication-algorithm ao;
            authentication-key-chain ao_hmac_chain;
        }
    }
    group ebgp_nokia_v6 {
        type external;
        local-address 2403:8100:1002:103::111;
        peer-as 38016;
        local-as 10458;
        tcp-mss 1450;
        neighbor 2406:c800:e000:1::2 {
            authentication-algorithm ao;
            authentication-key-chain ao_aes_chain;
        }
    }
}
```
