# Introduction

This document describes the basic steps needed in order to configure a BGP session using TCP-AO to secure the session.  This MD-CLI configuration example is applicable to Nokia SR OS 16.0.R15, 19.10.R7, 20.5.R1 and newer versions.  The classic CLI configuration is similar.

Enabling TCP-AO is quite simple and has two steps: 
1. Configure the key
2. Configure the BGP neighbor

More information can be found in the [System Management Guide](https://infocenter.nokia.com/public/7750SR205R1A/topic/com.sr.system.mgmt/html/security.html?cp=21_1_4_10#CHDGCGGD).

Note: The Unix `date --iso-8601=seconds` command output can be used as the begin time, if you want the key to be valid now.

# MD-CLI HMAC-SHA-1-96 Key Configuration

This example create an HMAC-SHA-1-96 key.  Replace `<plain-text>` with the key in plain text.

```
/configure system security keychains keychain "hmac-sha-1-96 keychain" tcp-option-number receive tcp-ao
/configure system security keychains keychain "hmac-sha-1-96 keychain" tcp-option-number send tcp-ao
/configure system security keychains keychain "hmac-sha-1-96 keychain" receive entry 9 authentication-key <plain-text>
/configure system security keychains keychain "hmac-sha-1-96 keychain" receive entry 9 algorithm hmac-sha-1-96
/configure system security keychains keychain "hmac-sha-1-96 keychain" receive entry 9 begin-time 2020-07-29T15:47:03-0400
/configure system security keychains keychain "hmac-sha-1-96 keychain" send entry 2 authentication-key <plain-text>
/configure system security keychains keychain "hmac-sha-1-96 keychain" send entry 2 algorithm hmac-sha-1-96
/configure system security keychains keychain "hmac-sha-1-96 keychain" send entry 2 begin-time 2020-07-29T15:47:03-0400
```

# MD-CLI AES-128-CMAC-96 Key Configuration

This example create an AES-128-CMAC-96 key.  Replace `<plain-text>` with the key in plain text.

```
/configure system security keychains keychain "aes-128-cmac-96 keychain" tcp-option-number receive tcp-ao
/configure system security keychains keychain "aes-128-cmac-96 keychain" tcp-option-number send tcp-ao
/configure system security keychains keychain "aes-128-cmac-96 keychain" receive entry 9 authentication-key <plain-text>
/configure system security keychains keychain "aes-128-cmac-96 keychain" receive entry 9 algorithm aes-128-cmac-96
/configure system security keychains keychain "aes-128-cmac-96 keychain" receive entry 9 begin-time 2020-07-29T15:47:03-0400
/configure system security keychains keychain "aes-128-cmac-96 keychain" send entry 2 authentication-key <plain-text>
/configure system security keychains keychain "aes-128-cmac-96 keychain" send entry 2 algorithm aes-128-cmac-96
/configure system security keychains keychain "aes-128-cmac-96 keychain" send entry 2 begin-time 2020-07-29T15:47:03-0400
```

# MD-CLI BGP Neighbor Configuration

Next, add the key to the BGP session.  This step is the same for IPv4 and IPv6 neighbors, and could also be added to the group instead of to the neighbor.

```
/configure router "Base" bgp neighbor <ip-address> authentication-keychain <keychain>
```

# Show Command Output

The following show command displays the configured BGP neighbor keychains.

```
[]
A:grhankin@er1-nyc# show router bgp auth-keychain "interoptest-aes"

===============================================================================
Sessions using key chain: interoptest-aes
===============================================================================
Group
    Peer address                        KeyChain name
-------------------------------------------------------------------------------
Juniper TCP-AO Test
    116.197.187.117                     interoptest-aes
Juniper TCP-AO Test
    2403:8100:1002:103::111             interoptest-aes
===============================================================================
```

# Complete Configuration Example and Output from June 2020 Interoperability Test with Juniper

```
configure {
    system {
        security {
            keychains {
                keychain "interoptest" {
                    tcp-option-number {
                        receive tcp-ao
                        send tcp-ao
                    }
                    receive {
                        entry 9 {
                            authentication-key "yzClLKIFsAVR91AobUXUT/ppPzL7bVxBrNNg" hash
                            algorithm hmac-sha-1-96
                            begin-time 2020-01-06T22:35:59.0Z
                        }
                    }
                    send {
                        entry 2 {
                            authentication-key "yzClLKIFsAVR91AobUXUT/ppPzL7bVxBrNNg" hash
                            algorithm hmac-sha-1-96
                            begin-time 2020-01-06T22:35:59.0Z
                        }
                    }
                }
                keychain "interoptest-aes" {
                    tcp-option-number {
                        receive tcp-ao
                        send tcp-ao
                    }
                    receive {
                        entry 9 {
                            authentication-key "yzClLKIFsAVR91AobUXUT/ppPzL7bVxBrNNg" hash
                            algorithm aes-128-cmac-96
                            begin-time 2020-06-09T04:00:00.0Z
                        }
                    }
                    send {
                        entry 2 {
                            authentication-key "yzClLKIFsAVR91AobUXUT/ppPzL7bVxBrNNg" hash
                            algorithm aes-128-cmac-96
                            begin-time 2020-06-09T04:00:00.0Z
                        }
                    }
                }
            }
        }
    }
    policy-options {
        policy-statement "RP_EXPORT_10458" {
            entry 10 {
                from {
                    protocol {
                        name [direct]
                    }
                }
                to {
                    protocol {
                        name [bgp]
                    }
                }
                action {
                    action-type accept
                }
            }
        }
    }
    router "Base" {
        bgp {
            group "Juniper TCP-AO Test" {
            }
            neighbor "116.197.187.117" {
                authentication-keychain "interoptest-aes"
                group "Juniper TCP-AO Test"
                multihop 255
                local-address 124.252.255.66
                peer-as 10458
                family {
                    ipv4 true
                }
                export {
                    policy ["RP_EXPORT_10458"]
                }
            }
            neighbor "2403:8100:1002:103::111" {
                authentication-keychain "interoptest-aes"
                group "Juniper TCP-AO Test"
                multihop 255
                local-address 2406:c800:e000:1::2
                peer-as 10458
                family {
                    ipv6 true
                }
                export {
                    policy ["RP_EXPORT_10458"]
                }
            }
        }
    }
}

[]
A:grhankin@er1-nyc# show router bgp summary all

===============================================================================
BGP Summary
===============================================================================
Legend : D - Dynamic Neighbor
===============================================================================
Neighbor
Description
ServiceId          AS PktRcvd InQ  Up/Down   State|Rcv/Act/Sent (Addr Family)
                      PktSent OutQ
-------------------------------------------------------------------------------
116.197.187.117
Def. Instance  10458     4784    0 01d11h59m 1/0/3 (IPv4)
                         4323    0
2403:8100:1002:103::111
Def. Instance  10458     3046    0 22h54m14s 0/0/2 (IPv6)
                         2754    0

-------------------------------------------------------------------------------
```
