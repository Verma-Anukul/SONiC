# OpenConfig support for Route Map

# High Level Design Document
#### Rev 0.1

# Table of Contents
  * [List of Tables](#list-of-tables)
  * [Revision](#revision)
  * [About This Manual](#about-this-manual)
  * [Scope](#scope)
  * [Definition/Abbreviation](#definitionabbreviation)
  * [1 Feature Overview](#1-feature-overview)
    * [1.1 Requirements](#11-requirements)
      * [1.1.1 Functional Requirements](#111-functional-requirements)
      * [1.1.2 Configuration and Management Requirements](#112-configuration-and-management-requirements)
    * [1.2 Design Overview](#12-design-overview)
      * [1.2.1 Basic Approach](#121-basic-approach)
      * [1.2.2 Container](#122-container)
  * [2 Functionality](#2-functionality)
    * [2.1 Target Deployment Use Cases](#21-target-deployment-use-cases)
  * [3 Design](#3-design)
    * [3.1 Overview](#31-overview)
    * [3.2 DB Changes](#32-db-changes)
      * [3.2.1 CONFIG DB](#321-config-db)
      * [3.2.2 APPL_DB](#322-appl_db)
      * [3.2.3 STATE DB](#323-state-db)
      * [3.2.4 ASIC DB](#324-asic-db)
      * [3.2.5 COUNTER DB](#325-counter-db)
    * [3.3 User Interface](#33-user-interface)
      * [3.3.1 Data Models](#331-data-models)
      * [3.3.2 OpenConfig Extensions](#332-openconfig-extensions)
      * [3.3.3 REST API Support](#333-rest-api-support)
      * [3.3.4 gNMI Support](#334-gnmi-support)
  * [4 Error Handling](#4-error-handling)
  * [5 Unit Test Cases](#5-unit-test-cases)
    * [5.1 Functional Test Cases](#51-functional-test-cases)
    * [5.2 Negative Test Cases](#52-negative-test-cases)
  * [6 References](#6-references)
  
# List of Tables
[Table 1: Abbreviations](#table-1-abbreviations)

[Table 2: OpenConfig YANG to SONiC YANG Mapping](#table-2-openconfig-yang-to-sonic-yang-mapping)

# Revision
| Rev |     Date    |       Author          | Change Description                |
|:---:|:-----------:|:---------------------:|-----------------------------------|
| 0.1 | 02/26/2026  | Anukul Verma          | Initial version                   |

# About this Manual
This document provides general information about the OpenConfig configuration of Route Map (Policy Definitions) in SONiC.

# Scope
- This document describes the high level design of Route Map configuration using OpenConfig models via REST & gNMI.
- This does not cover the SONiC KLISH CLI.
- This covers policy-definitions container with statements, conditions, and actions.
- Supported attributes in OpenConfig YANG tree:

```yang
module: openconfig-routing-policy
  +--rw routing-policy
     +--rw policy-definitions
        +--rw policy-definition* [name]
           +--rw name          -> ../config/name
           +--rw config
           |  +--rw name?   string
           +--ro state
           |  +--ro name?   string
           +--rw statements
              +--rw statement* [name]
                 +--rw name          -> ../config/name
                 +--rw config
                 |  +--rw name?                    string
                 |  +--rw oc-rp-ext:description?   string
                 +--ro state
                 |  +--ro name?                    string
                 |  +--ro oc-rp-ext:description?   string
                 +--rw conditions
                 |  +--rw match-interface
                 |  |  +--rw config
                 |  |  |  +--rw interface?   -> /oc-if:interfaces/interface/name
                 |  |  +--ro state
                 |  |     +--ro interface?   -> /oc-if:interfaces/interface/name
                 |  +--rw match-prefix-set
                 |  |  +--rw config
                 |  |  |  +--rw prefix-set?   -> ../../../../../../../../defined-sets/prefix-sets/prefix-set/config/name
                 |  |  +--ro state
                 |  |     +--ro prefix-set?   -> ../../../../../../../../defined-sets/prefix-sets/prefix-set/config/name
                 |  +--rw match-tag-set
                 |  |  +--rw config
                 |  |  |  +--rw tag-set?             -> ../../../../../../../../defined-sets/tag-sets/tag-set/name
                 |  |  |  +--rw match-set-options?   oc-pol-types:match-set-options-restricted-type
                 |  |  +--ro state
                 |  |     +--ro tag-set?             -> ../../../../../../../../defined-sets/tag-sets/tag-set/name
                 |  |     +--ro match-set-options?   oc-pol-types:match-set-options-restricted-type
                 |  +--rw oc-bgp-pol:bgp-conditions
                 |     +--rw oc-bgp-pol:config
                 |     +--ro oc-bgp-pol:state
                 |     +--rw oc-bgp-pol:ext-community-count
                 |     |  +--rw oc-bgp-pol:config
                 |     |  |  +--rw oc-bgp-pol:operator?   identityref
                 |     |  |  +--rw oc-bgp-pol:value?      uint32
                 |     |  +--ro oc-bgp-pol:state
                 |     |     +--ro oc-bgp-pol:operator?   identityref
                 |     |     +--ro oc-bgp-pol:value?      uint32
                 |     +--rw oc-bgp-pol:match-community-set
                 |        +--rw oc-bgp-pol:config
                 |        |  +--rw oc-bgp-pol:community-set?   -> /oc-rpol:routing-policy/defined-sets/oc-bgp-pol:bgp-defined-sets/community-sets/community-set/community-set-name
                 |        +--ro oc-bgp-pol:state
                 |           +--ro oc-bgp-pol:community-set?   -> /oc-rpol:routing-policy/defined-sets/oc-bgp-pol:bgp-defined-sets/community-sets/community-set/community-set-name
                 +--rw actions
                    +--rw config
                    |  +--rw policy-result?                       policy-result-type
                    |  +--rw oc-rp-ext:next-statement?            uint16
                    |  +--rw oc-rp-ext:on-match-next?             boolean
                    |  +--rw oc-rp-ext:on-match-goto-statement?   uint16
                    +--ro state
                    |  +--ro policy-result?                       policy-result-type
                    |  +--ro oc-rp-ext:next-statement?            uint16
                    |  +--ro oc-rp-ext:on-match-next?             boolean
                    |  +--ro oc-rp-ext:on-match-goto-statement?   uint16
                    +--rw oc-bgp-pol:bgp-actions
                       +--rw oc-bgp-pol:config
                       |  +--rw oc-bgp-pol:set-next-hop?        bgp-next-hop-type
                       |  +--rw oc-rp-ext:set-source-address?   oc-inet:ip-address
                       +--ro oc-bgp-pol:state
                       |  +--ro oc-bgp-pol:set-next-hop?        bgp-next-hop-type
                       |  +--ro oc-rp-ext:set-source-address?   oc-inet:ip-address
                       +--rw oc-bgp-pol:set-as-path-prepend
                       |  +--rw oc-bgp-pol:config
                       |  |  +--rw oc-bgp-pol:repeat-n?      uint8
                       |  |  +--rw oc-bgp-pol:asn?           oc-inet:as-number
                       |  |  +--rw oc-rp-ext:asn-sequence?   string
                       |  +--ro oc-bgp-pol:state
                       |     +--ro oc-bgp-pol:repeat-n?      uint8
                       |     +--ro oc-bgp-pol:asn?           oc-inet:as-number
                       |     +--ro oc-rp-ext:asn-sequence?   string
                       +--rw oc-bgp-pol:set-community
                       |  +--rw oc-bgp-pol:config
                       |  |  +--rw oc-bgp-pol:method?    enumeration
                       |  |  +--rw oc-bgp-pol:options?   bgp-set-community-option-type
                       |  +--ro oc-bgp-pol:state
                       |  |  +--ro oc-bgp-pol:method?    enumeration
                       |  |  +--ro oc-bgp-pol:options?   bgp-set-community-option-type
                       |  +--rw oc-bgp-pol:inline
                       |     +--rw oc-bgp-pol:config
                       |     |  +--rw oc-bgp-pol:communities*   union
                       |     +--ro oc-bgp-pol:state
                       |        +--ro oc-bgp-pol:communities*   union
                       +--rw oc-rp-ext:set-large-community
                          +--rw oc-rp-ext:config
                          |  +--rw oc-rp-ext:method?    enumeration
                          |  +--rw oc-rp-ext:options?   bgp-set-large-community-option-type
                          +--ro oc-rp-ext:state
                          |  +--ro oc-rp-ext:method?    enumeration
                          |  +--ro oc-rp-ext:options?   bgp-set-large-community-option-type
                          +--rw oc-rp-ext:inline
                             +--rw oc-rp-ext:config
                             |  +--rw oc-rp-ext:large-communities*   string
                             +--ro oc-rp-ext:state
                                +--ro oc-rp-ext:large-communities*   string
```

# Definition/Abbreviation
### Table 1: Abbreviations
| **Term**                 | **Definition**                         |
|--------------------------|-------------------------------------|
| YANG                     | Yet Another Next Generation: modular language representing data structures in an XML tree format        |
| REST                     | REpresentative State Transfer |
| gNMI                     | gRPC Network Management Interface: used to retrieve or manipulate the state of a device via telemetry or configuration data         |
| BGP                      | Border Gateway Protocol   |
| AS                       | Autonomous System   |

# 1 Feature Overview
## 1.1 Requirements
### 1.1.1 Functional Requirements
1. Provide support for OpenConfig YANG models for Route Map (Policy Definition) configuration.
2. Configure/Set, GET, and Delete route map parameters.
3. Support configuration of policy definitions with multiple statements.
4. Support match conditions: interface, prefix-set, tag-set, community-set.
5. Support actions: permit/deny, next-hop, AS path prepend, community operations.
6. Support statement ordering and flow control (next, goto).
7. Support BGP-specific conditions and actions.
8. Map OpenConfig routing-policy model to SONiC CONFIG_DB tables.

### 1.1.2 Configuration and Management Requirements
The Route Map configurations can be done via REST and gNMI. The implementation will return an error if a configuration is not allowed.

## 1.2 Design Overview
### 1.2.1 Basic Approach
SONiC already supports Route Map configurations via CLI. This feature adds support for OpenConfig based YANG models using transformer based implementation in the Management Framework.

The implementation provides mapping between:
- OpenConfig policy-definition config parameters → SONiC ROUTE_MAP tables
- OpenConfig statement conditions → SONiC match criteria
- OpenConfig statement actions → SONiC set actions

### 1.2.2 Container
The code changes for this feature are part of *mgmt-framework* container which includes the REST server and *gnmi* container for gNMI support in *sonic-mgmt-common* repository.

# 2 Functionality
## 2.1 Target Deployment Use Cases
1. REST client through which the user can perform PATCH, DELETE, POST, PUT, and GET operations on Route Map configuration paths.
2. gNMI client with support for capabilities, get, set, and subscribe operations based on the supported YANG models.

# 3 Design
## 3.1 Overview
This HLD design is in line with the [Management Framework HLD](https://github.com/project-arlo/SONiC/blob/master/doc/mgmt/Management%20Framework.md)

The implementation uses transformer functions in `translib/transformer/xfmr_route_map.go` to map between OpenConfig and SONiC data models.

### 3.1.1 Mapping Details

#### Table 2: OpenConfig YANG to SONiC YANG Mapping

| OpenConfig YANG Node | SONiC YANG File | DB Name | Table:Field |
|---------------------|-----------------|---------|-------------|
| **policy-definition** | | | |
| config/name | sonic-route-map.yang | CONFIG_DB | ROUTE_MAP:`<policy-name>\|<statement-name>` (key) |
| config/name | sonic-route-map.yang | CONFIG_DB | ROUTE_MAP_SET:name (policy-definition name) |
| **statements/statement** | | | |
| config/name | sonic-route-map.yang | CONFIG_DB | ROUTE_MAP:`<policy-name>\|<statement-name>` |
| config/description | sonic-route-map.yang | CONFIG_DB | ROUTE_MAP:description |
| **conditions/match-interface** | | | |
| config/interface | sonic-route-map.yang | CONFIG_DB | ROUTE_MAP:match_interface |
| **conditions/match-prefix-set** | | | |
| config/prefix-set | sonic-route-map.yang | CONFIG_DB | ROUTE_MAP:match_prefix_set |
| **conditions/match-tag-set** | | | |
| config/tag-set | sonic-route-map.yang | CONFIG_DB | ROUTE_MAP:match_tag |
| config/match-set-options | sonic-route-map.yang | CONFIG_DB | Not supported |
| **conditions/bgp-conditions/match-community-set** | | | |
| config/community-set | sonic-route-map.yang | CONFIG_DB | ROUTE_MAP:match_community |
| **conditions/bgp-conditions/ext-community-count** | | | |
| config/operator | sonic-route-map.yang | CONFIG_DB | Not fully supported |
| config/value | sonic-route-map.yang | CONFIG_DB | Not fully supported |
| **actions** | | | |
| config/policy-result | sonic-route-map.yang | CONFIG_DB | ROUTE_MAP:route_operation |
| config/next-statement | sonic-route-map.yang | CONFIG_DB | ROUTE_MAP:next_statement |
| config/on-match-next | sonic-route-map.yang | CONFIG_DB | ROUTE_MAP:on_match_next |
| config/on-match-goto-statement | sonic-route-map.yang | CONFIG_DB | ROUTE_MAP:on_match_goto_statement |
| **actions/bgp-actions** | | | |
| config/set-next-hop | sonic-route-map.yang | CONFIG_DB | ROUTE_MAP:set_next_hop (IPv4) / ROUTE_MAP:set_ipv6_next_hop_global (IPv6) |
| config/set-source-address | sonic-route-map.yang | CONFIG_DB | ROUTE_MAP_SET:set_src |
| **actions/bgp-actions/set-as-path-prepend** | | | |
| config/repeat-n | sonic-route-map.yang | CONFIG_DB | ROUTE_MAP_SET:set_repeat_asn |
| config/asn | sonic-route-map.yang | CONFIG_DB | ROUTE_MAP_SET:set_asn |
| config/asn-sequence | sonic-route-map.yang | CONFIG_DB | ROUTE_MAP_SET:set_asn_list |
| **actions/bgp-actions/set-community** | | | |
| inline/config/communities | sonic-route-map.yang | CONFIG_DB | ROUTE_MAP_SET:community |
| config/options | sonic-route-map.yang | CONFIG_DB | ROUTE_MAP_SET:comm_list_delete (delete only) |
| **actions/bgp-actions/set-large-community** | | | |
| inline/config/large-communities | sonic-route-map.yang | CONFIG_DB | ROUTE_MAP_SET:large_community |

**Notes:**
- **Bold** entries indicate major feature categories/containers
- State nodes mirror their corresponding config nodes and are read-only
- Key format for ROUTE_MAP entries: `"<policy-name>|<statement-name>"`
- Statement names represent sequence numbers in SONiC
- **Set next-hop:** Stored in ROUTE_MAP table (fields `set_next_hop` for IPv4, `set_ipv6_next_hop_global` for IPv6), not ROUTE_MAP_SET
- ROUTE_MAP_SET table is used for other BGP-specific set actions (set-source-address, community, AS path, etc.)
- match-tag-set/match-set-options: Only ANY/ALL supported, INVERT not supported
- ext-community-count: Limited support in current implementation

## 3.2 DB Changes

### 3.2.1 CONFIG DB

The implementation uses existing SONiC YANG schemas for Route Map configuration.

**Note:** Most BGP set actions (set-source-address, set-community, AS path prepend, etc.) are stored in the ROUTE_MAP_SET table, while match conditions, basic actions, and set-next-hop are in the ROUTE_MAP table. The set-next-hop fields are `set_next_hop` (IPv4) and `set_ipv6_next_hop_global` (IPv6) in ROUTE_MAP.

### 3.2.2 APPL_DB
There are no changes to APPL_DB schema definition for this feature.

### 3.2.3 STATE DB
There are no changes to STATE DB schema definition for this feature.

### 3.2.4 ASIC DB
There are no changes to ASIC DB schema definition.

### 3.2.5 COUNTER DB
There are no changes to COUNTER DB schema definition.

## 3.3 User Interface
### 3.3.1 Data Models
The implementation uses standard OpenConfig YANG models:
- **openconfig-routing-policy.yang**: Main routing policy model
- **openconfig-bgp-policy.yang**: BGP-specific policy definitions

### 3.3.2 OpenConfig Extensions

The following extensions are added to the standard OpenConfig routing-policy model in **openconfig-routing-policy-ext.yang**:

#### Statement Description Extension
```yang
augment /oc-rpol:routing-policy/oc-rpol:policy-definitions/oc-rpol:policy-definition/oc-rpol:statements/oc-rpol:statement/oc-rpol:config {
  leaf description {
    type string;
    description "Description for the route-map statement";
  }
}
```

#### Flow Control Extensions
```yang
augment /oc-rpol:routing-policy/oc-rpol:policy-definitions/oc-rpol:policy-definition/oc-rpol:statements/oc-rpol:statement/oc-rpol:actions/oc-rpol:config {
  leaf next-statement {
    type uint16;
    description "Jump to a specific statement number";
  }
  
  leaf on-match-next {
    type boolean;
    description "On match, continue to next statement";
  }
  
  leaf on-match-goto-statement {
    type uint16;
    description "On match, jump to specific statement";
  }
}
```

#### BGP Source Address Extension
```yang
augment /oc-rpol:routing-policy/oc-rpol:policy-definitions/oc-rpol:policy-definition/oc-rpol:statements/oc-rpol:statement/oc-rpol:actions/oc-bgp-pol:bgp-actions/oc-bgp-pol:config {
  leaf set-source-address {
    type oc-inet:ip-address;
    description "Set the source IP address";
  }
}
```

#### AS Path Prepend Sequence Extension
```yang
augment /oc-rpol:routing-policy/oc-rpol:policy-definitions/oc-rpol:policy-definition/oc-rpol:statements/oc-rpol:statement/oc-rpol:actions/oc-bgp-pol:bgp-actions/oc-bgp-pol:set-as-path-prepend/oc-bgp-pol:config {
  leaf asn-sequence {
    type string;
    description "Sequence of ASNs to prepend (space-separated)";
  }
}
```

#### Large Community Extensions
```yang
augment /oc-rpol:routing-policy/oc-rpol:policy-definitions/oc-rpol:policy-definition/oc-rpol:statements/oc-rpol:statement/oc-rpol:actions/oc-bgp-pol:bgp-actions {
  container set-large-community {
    container config {
      leaf method {
        type enumeration {
          enum INLINE;
          enum REFERENCE;
        }
      }
      leaf options {
        type bgp-set-large-community-option-type;
      }
    }
    container inline {
      leaf-list large-communities {
        type string;
      }
    }
  }
}
```

### 3.3.3 REST API Support

#### 3.3.3.1 GET
Supported at all levels (container, list, and leaf).

**Example: GET route map config**
```bash
curl -X GET -k "https://192.168.1.1/restconf/data/openconfig-routing-policy:routing-policy/policy-definitions/policy-definition=RM1" -H "accept: application/yang-data+json"
```

Response:
```json
{
  "openconfig-routing-policy:policy-definition": [
    {
      "name": "RM1",
      "config": {
        "name": "RM1"
      },
      "statements": {
        "statement": [
          {
            "name": "10",
            "config": {
              "name": "10",
              "openconfig-routing-policy-ext:description": "First statement"
            },
            "conditions": {
              "match-prefix-set": {
                "config": {
                  "prefix-set": "PREFIX1"
                }
              }
            },
            "actions": {
              "config": {
                "policy-result": "ACCEPT_ROUTE"
              },
              "openconfig-bgp-policy:bgp-actions": {
                "config": {
                  "set-next-hop": "10.0.0.1"
                }
              }
            }
          }
        ]
      }
    }
  ]
}
```

#### 3.3.3.2 POST
Used to create new configuration. Supported at container and leaf levels.

**Example: POST a new route map with statement**
```bash
curl -X POST -k "https://192.168.1.1/restconf/data/openconfig-routing-policy:routing-policy/policy-definitions" \
  -H "accept: */*" \
  -H "Content-Type: application/yang-data+json" \
  -d '{
    "policy-definition": [
      {
        "name": "RM1",
        "config": {
          "name": "RM1"
        },
        "statements": {
          "statement": [
            {
              "name": "10",
              "config": {
                "name": "10",
                "openconfig-routing-policy-ext:description": "Match prefix and set next-hop"
              },
              "conditions": {
                "match-prefix-set": {
                  "config": {
                    "prefix-set": "PREFIX1"
                  }
                }
              },
              "actions": {
                "config": {
                  "policy-result": "ACCEPT_ROUTE"
                },
                "openconfig-bgp-policy:bgp-actions": {
                  "config": {
                    "set-next-hop": "10.0.0.1"
                  }
                }
              }
            }
          ]
        }
      }
    ]
  }'
```

#### 3.3.3.3 PUT
Used to replace entire configuration. Supported at container and leaf levels.

**Example: PUT a route map statement**
```bash
curl -X PUT -k "https://192.168.1.1/restconf/data/openconfig-routing-policy:routing-policy/policy-definitions/policy-definition=RM1/statements/statement=10" \
  -H "accept: */*" \
  -H "Content-Type: application/yang-data+json" \
  -d '{
    "statement": [
      {
        "name": "10",
        "config": {
          "name": "10"
        },
        "conditions": {
          "openconfig-bgp-policy:bgp-conditions": {
            "match-community-set": {
              "config": {
                "community-set": "COMM1"
              }
            }
          }
        },
        "actions": {
          "config": {
            "policy-result": "ACCEPT_ROUTE"
          }
        }
      }
    ]
  }'
```

#### 3.3.3.4 PATCH
Used to modify specific fields without replacing entire configuration. Supported at all levels.

**Example: PATCH AS path prepend action**
```bash
curl -X PATCH -k "https://192.168.1.1/restconf/data/openconfig-routing-policy:routing-policy/policy-definitions/policy-definition=RM1/statements/statement=10/actions/openconfig-bgp-policy:bgp-actions/set-as-path-prepend/config" \
  -H "accept: */*" \
  -H "Content-Type: application/yang-data+json" \
  -d '{
    "openconfig-bgp-policy:config": {
      "asn": 65100,
      "repeat-n": 3
    }
  }'
```

**Example: PATCH community action (extension)**
```bash
curl -X PATCH -k "https://192.168.1.1/restconf/data/openconfig-routing-policy:routing-policy/policy-definitions/policy-definition=RM1/statements/statement=10/actions/openconfig-bgp-policy:bgp-actions/set-community" \
  -H "accept: */*" \
  -H "Content-Type: application/yang-data+json" \
  -d '{
    "openconfig-bgp-policy:set-community": {
      "config": {
        "method": "INLINE",
        "options": "ADD"
      },
      "inline": {
        "config": {
          "communities": ["65000:100", "65000:200"]
        }
      }
    }
  }'
```

#### 3.3.3.5 DELETE
Supported at all levels.

**Example: DELETE route map statement**
```bash
curl -X DELETE -k "https://192.168.1.1/restconf/data/openconfig-routing-policy:routing-policy/policy-definitions/policy-definition=RM1/statements/statement=10" \
  -H "accept: */*"
```

**Example: DELETE entire route map**
```bash
curl -X DELETE -k "https://192.168.1.1/restconf/data/openconfig-routing-policy:routing-policy/policy-definitions/policy-definition=RM1" \
  -H "accept: */*"
```

### 3.3.4 gNMI Support

#### 3.3.4.1 GET

**Example: GET route map**
```bash
gnmi_get -insecure -logtostderr -username admin -password password \
  -target_addr localhost:8080 \
  -xpath /openconfig-routing-policy:routing-policy/policy-definitions/policy-definition[name=RM1]
```

#### 3.3.4.2 SET

**Example: SET route map statement with actions**
```bash
gnmi_set -insecure -logtostderr -username admin -password password \
  -target_addr localhost:8080 \
  -xpath_target OC-YANG \
  -update "/openconfig-routing-policy:routing-policy/policy-definitions/policy-definition[name=RM1]/statements/statement[name=10]/actions/bgp-actions/config/set-next-hop:10.0.0.1"
```

#### 3.3.4.3 DELETE

**Example: DELETE route map**
```bash
gnmi_set -insecure -logtostderr -username admin -password password \
  -target_addr localhost:8080 \
  -xpath_target OC-YANG \
  -delete /openconfig-routing-policy:routing-policy/policy-definitions/policy-definition[name=RM1]
```

#### 3.3.4.4 SUBSCRIBE

gNMI subscription is supported for monitoring configuration changes on route map parameters.

# 4 Error Handling

The implementation handles various error scenarios and returns appropriate error responses:

- **Route Map Not Found**: Attempting to configure on a non-existent route map will return an error.
- **Invalid Statement Name**: Providing an invalid statement name format will return an error.
- **Invalid Prefix Set Reference**: Referencing a non-existent prefix set will return an error.
- **Invalid Community Set Reference**: Referencing a non-existent community set will return an error.
- **Invalid Next-Hop Address**: Providing an invalid IP address will return an error.
- **Invalid ASN**: Providing an invalid AS number will return an error.
- **Duplicate Statement**: Attempting to create a duplicate statement will return an error.
- **Unsupported match-set-options**: INVERT option for match-tag-set is not supported - only ANY and ALL are supported.
- **Limited ext-community-count Support**: Extended community count operators may have limited support in current implementation.

# 5 Unit Test Cases

Comprehensive test cases are available in the transformer test suite.

## 5.1 Functional Test Cases

### 5.1.1 Policy Definition Configuration Tests

- POST route map with name
- GET route map config
- DELETE route map
- GET all route maps

### 5.1.2 Statement Configuration Tests

- POST statement with name and description (extension)
- GET all statements
- GET specific statement
- DELETE statement

### 5.1.3 Match Condition Tests

- POST statement with match-interface condition
- POST statement with match-prefix-set condition
- POST statement with match-tag-set condition
- POST statement with match-community-set condition (BGP)
- POST statement with ext-community-count condition (BGP)
- GET match conditions
- PATCH match conditions
- DELETE match conditions

### 5.1.4 Action Configuration Tests

- POST statement with policy-result (permit/deny)
- POST statement with next-hop action (BGP)
- POST statement with AS path prepend action (BGP)
- POST statement with AS path prepend sequence (extension)
- POST statement with community set action (BGP)
- POST statement with large community action (extension)
- POST statement with source address action (extension)
- GET actions
- PATCH actions
- DELETE actions

### 5.1.5 Flow Control Tests

- POST statement with next-statement (extension)
- POST statement with on-match-next (extension)
- POST statement with on-match-goto-statement (extension)
- GET flow control config
- PATCH flow control config

### 5.1.6 Integration Tests

- Configure route map with multiple statements
- Configure statement with multiple match conditions
- Configure statement with multiple actions
- GET entire policy-definitions container
- Configure complex BGP policy with communities and AS path

## 5.2 Negative Test Cases

- POST route map with empty name
- POST duplicate route map
- POST statement with non-existent prefix set reference
- POST statement with non-existent community set reference
- POST statement with invalid next-hop address
- POST statement with invalid ASN
- PATCH non-existing route map
- DELETE non-existing statement
- POST statement with invalid policy-result value

# 6 References

1. [OpenConfig Routing Policy YANG Model](https://github.com/openconfig/public/tree/master/release/models/policy)
2. [OpenConfig BGP Policy YANG Model](https://github.com/openconfig/public/tree/master/release/models/bgp)
3. [SONiC Management Framework HLD](https://github.com/project-arlo/SONiC/blob/master/doc/mgmt/Management%20Framework.md)
4. [SONiC Route Map Configuration](https://github.com/sonic-net/SONiC/blob/master/doc/routing/route-map.md)
