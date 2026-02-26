# OpenConfig support for Static Routes

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
This document provides general information about the OpenConfig configuration of static routes in SONiC.

# Scope
- This document describes the high level design of static route configuration using OpenConfig models via REST & gNMI.
- This does not cover the SONiC KLISH CLI.
- This covers IPv4 and IPv6 static route configuration with next-hops, including BFD enablement and VRF support.
- Supported attributes in OpenConfig YANG tree:

```yang
module: openconfig-network-instance
  +--rw network-instances
     +--rw network-instance* [name]
        +--rw name                                        -> ../config/name
        +--rw config
        |  +--rw name?   string
        |  +--rw type    identityref
        +--ro state
        |  +--ro name?   string
        |  +--ro type    identityref
        +--rw protocols
        |  +--rw protocol* [identifier name]
        |     +--rw identifier          -> ../config/identifier
        |     +--rw name                -> ../config/name
        |     +--rw config
        |     |  +--rw identifier?   identityref
        |     |  +--rw name?         string
        |     +--ro state
        |     |  +--ro identifier?   identityref
        |     |  +--ro name?         string
        |     +--rw static-routes
        |     |  +--rw static* [prefix]
        |     |     +--rw prefix       -> ../config/prefix
        |     |     +--rw config
        |     |     |  +--rw prefix?   inet:ip-prefix
        |     |     +--ro state
        |     |     |  +--ro prefix?   inet:ip-prefix
        |     |     +--rw next-hops
        |     |        +--rw next-hop* [index]
        |     |           +--rw index            -> ../config/index
        |     |           +--rw config
        |     |           |  +--rw index?                                   string
        |     |           |  +--rw next-hop?                                union
        |     |           |  +--rw oc-loc-rt-netinst:nh-network-instance?   -> /oc-ni:network-instances/network-instance/config/name
        |     |           +--ro state
        |     |           |  +--ro index?                                   string
        |     |           |  +--ro next-hop?                                union
        |     |           |  +--ro oc-loc-rt-netinst:nh-network-instance?   -> /oc-ni:network-instances/network-instance/config/name
        |     |           +--rw enable-bfd
        |     |           |  +--rw config
        |     |           |  |  +--rw enabled?   boolean
        |     |           |  +--ro state
        |     |           |     +--ro enabled?   boolean
        |     |           +--rw interface-ref
        |     |              +--rw config
        |     |              |  +--rw interface?      -> /oc-if:interfaces/interface/name
        |     |              |  +--rw subinterface?   -> /oc-if:interfaces/interface[oc-if:name=current()/../interface]/subinterfaces/subinterface/index
        |     |              +--ro state
        |     |                 +--ro interface?      -> /oc-if:interfaces/interface/name
        |     |                 +--ro subinterface?   -> /oc-if:interfaces/interface[oc-if:name=current()/../interface]/subinterfaces/subinterface/index
```

# Definition/Abbreviation
### Table 1: Abbreviations
| **Term**                 | **Definition**                         |
|--------------------------|-------------------------------------|
| YANG                     | Yet Another Next Generation: modular language representing data structures in an XML tree format        |
| REST                     | REpresentative State Transfer |
| gNMI                     | gRPC Network Management Interface: used to retrieve or manipulate the state of a device via telemetry or configuration data         |
| VRF                      | Virtual Routing and Forwarding   |
| BFD                      | Bidirectional Forwarding Detection   |
| NH                       | Next Hop   |

# 1 Feature Overview
## 1.1 Requirements
### 1.1.1 Functional Requirements
1. Provide support for OpenConfig YANG models for static route configuration.
2. Configure/Set, GET, and Delete static routes with IPv4 and IPv6 prefixes.
3. Support multiple next-hops per static route.
4. Support next-hop configuration with IP address or interface.
5. Support BFD enablement per next-hop.
6. Support VRF (network instance) for next-hop configuration.
7. Support interface binding for next-hops.
8. Map OpenConfig static-routes model to SONiC STATIC_ROUTE table.

### 1.1.2 Configuration and Management Requirements
The static route configurations can be done via REST and gNMI. The implementation will return an error if a configuration is not allowed.

**Important Notes:**
- Static routes are configured under network-instance/protocols/protocol with identifier=STATIC.
- Multiple next-hops can be configured per static route.
- BFD can be enabled per next-hop for fast failure detection.

## 1.2 Design Overview
### 1.2.1 Basic Approach
SONiC already supports static route configurations via FRR. This feature adds support for OpenConfig based YANG models using transformer based implementation in the Management Framework.

The implementation provides mapping between:
- OpenConfig static-routes config parameters → SONiC STATIC_ROUTE table
- OpenConfig next-hop parameters → SONiC STATIC_ROUTE table fields
- OpenConfig BFD configuration → SONiC BFD integration

### 1.2.2 Container
The code changes for this feature are part of *mgmt-framework* container which includes the REST server and *gnmi* container for gNMI support in *sonic-mgmt-common* repository.

# 2 Functionality
## 2.1 Target Deployment Use Cases
1. REST client through which the user can perform PATCH, DELETE, POST, PUT, and GET operations on static route configuration paths.
2. gNMI client with support for capabilities, get, set, and subscribe operations based on the supported YANG models.

# 3 Design
## 3.1 Overview
This HLD design is in line with the [Management Framework HLD](https://github.com/project-arlo/SONiC/blob/master/doc/mgmt/Management%20Framework.md)

The implementation uses transformer functions in `translib/transformer/xfmr_static_route.go` to map between OpenConfig and SONiC data models.

### 3.1.1 Mapping Details

#### Table 2: OpenConfig YANG to SONiC YANG Mapping

| OpenConfig YANG Node | SONiC YANG File | DB Name | Table:Field |
|---------------------|-----------------|---------|-------------|
| **protocols/protocol** | | | |
| config/identifier | N/A | N/A | Not stored - must be STATIC for static routes |
| config/name | N/A | N/A | Not stored - must be "static" |
| **static-routes/static** | | | |
| config/prefix | sonic-static-route.yang | CONFIG_DB | STATIC_ROUTE:vrf_name\|prefix (key) |
| **next-hops/next-hop** | | | |
| config/index | N/A | N/A | Not stored - index maps to nexthop array position |
| config/next-hop | sonic-static-route.yang | CONFIG_DB | STATIC_ROUTE:nexthop (comma-separated for multiple) |
| config/nh-network-instance | sonic-static-route.yang | CONFIG_DB | STATIC_ROUTE:nexthop-vrf (comma-separated for multiple) |
| **enable-bfd** | | | |
| config/enabled | sonic-static-route.yang | CONFIG_DB | STATIC_ROUTE:bfd (comma-separated for multiple) |
| **interface-ref** | | | |
| config/interface | sonic-static-route.yang | CONFIG_DB | STATIC_ROUTE:ifname (comma-separated for multiple) |

**Notes:**
- **Bold** entries indicate major feature categories/containers
- State nodes mirror their corresponding config nodes and are read-only
- **Key format for STATIC_ROUTE:** `"vrf_name|prefix"` (e.g., "default|172.16.0.0/24", "Vrf1|2001:db8::/32")
- Protocol identifier must be "STATIC" and name "static" - these are not stored in CONFIG_DB
- **Multiple next-hops:** Single STATIC_ROUTE entry per prefix with comma-separated values in nexthop, ifname, nexthop-vrf, and bfd fields
- Next-hop VRF field name is `nexthop-vrf` (hyphenated, not underscore)
- BFD field name is `bfd` (not `bfd_enable`)
- The index field is not stored - it represents the position in the nexthop array
- **Standard OpenConfig next-hop/config/network-instance is not supported** (deviated as not-supported in openconfig-network-instance-deviation.yang)
- The `nh-network-instance` extension field maps to STATIC_ROUTE:nexthop-vrf for VRF route leaking

## 3.2 DB Changes

### 3.2.1 CONFIG DB

The **sonic-static-route.yang** schema is used for static route configuration.

**Note:** The STATIC_ROUTE table does not store protocol identifier or name fields - these are implied by the table itself and must be "STATIC" and "static" respectively in OpenConfig paths.

**FRR Configuration:**

Changes are made in FRR configuration to support static routes with BFD. The frrcfgd daemon subscribes to STATIC_ROUTE table and configures corresponding FRR static route commands.

### 3.2.2 APPL_DB
Static route information is pushed to APPL_DB ROUTE_TABLE by the routing protocol daemon.

### 3.2.3 STATE DB
There are no changes to STATE DB schema definition for this feature.

### 3.2.4 ASIC DB
There are no changes to ASIC DB schema definition.

### 3.2.5 COUNTER DB
There are no changes to COUNTER DB schema definition.

## 3.3 User Interface
### 3.3.1 Data Models
The implementation uses standard OpenConfig YANG models:
- **openconfig-network-instance.yang**: Main network instance model
- **openconfig-local-routing.yang**: Static route definitions
- **openconfig-policy-types.yang**: Protocol identifier types
- **openconfig-interfaces.yang**: Interface references

### 3.3.2 OpenConfig Extensions

This implementation uses the standard OpenConfig local-routing extension for next-hop network instance:

```yang
import openconfig-local-routing-ext { prefix oc-loc-rt-netinst; }

augment /oc-ni:network-instances/oc-ni:network-instance/oc-ni:protocols/
        oc-ni:protocol/oc-ni:static-routes/oc-ni:static/oc-ni:next-hops/
        oc-ni:next-hop/oc-ni:config {
  leaf nh-network-instance {
    type leafref {
      path /oc-ni:network-instances/oc-ni:network-instance/oc-ni:config/oc-ni:name;
    }
    description "Next-hop VRF/network-instance";
  }
}
```

**Purpose**: Allows specifying a different VRF (network instance) for the next-hop, enabling inter-VRF route leaking.

**Usage**: Reference another network instance name for cross-VRF routing.

### 3.3.3 REST API Support

#### 3.3.3.1 GET
Supported at all levels (container, list, and leaf).

**Example: GET all static routes**
```bash
curl -X GET -k "https://192.168.1.1/restconf/data/openconfig-network-instance:network-instances/network-instance=default/protocols/protocol=STATIC,static/static-routes" -H "accept: application/yang-data+json"
```

Response:
```json
{
  "openconfig-local-routing:static-routes": {
    "static": [
      {
        "prefix": "10.1.1.0/24",
        "config": {
          "prefix": "10.1.1.0/24"
        },
        "next-hops": {
          "next-hop": [
            {
              "index": "1",
              "config": {
                "index": "1",
                "next-hop": "10.0.0.1"
              }
            }
          ]
        }
      }
    ]
  }
}
```

**Example: GET specific static route**
```bash
curl -X GET -k "https://192.168.1.1/restconf/data/openconfig-network-instance:network-instances/network-instance=default/protocols/protocol=STATIC,static/static-routes/static=10.1.1.0%2F24" -H "accept: application/yang-data+json"
```

#### 3.3.3.2 POST
Used to create new configuration. Supported at container and leaf levels.

**Example: POST a new static route with next-hop**
```bash
curl -X POST -k "https://192.168.1.1/restconf/data/openconfig-network-instance:network-instances/network-instance=default/protocols/protocol=STATIC,static/static-routes" \
  -H "accept: */*" \
  -H "Content-Type: application/yang-data+json" \
  -d '{
    "static": [
      {
        "prefix": "10.1.1.0/24",
        "config": {
          "prefix": "10.1.1.0/24"
        },
        "next-hops": {
          "next-hop": [
            {
              "index": "1",
              "config": {
                "index": "1",
                "next-hop": "10.0.0.1"
              },
              "enable-bfd": {
                "config": {
                  "enabled": true
                }
              }
            }
          ]
        }
      }
    ]
  }'
```

**Example: POST static route with interface next-hop**
```bash
curl -X POST -k "https://192.168.1.1/restconf/data/openconfig-network-instance:network-instances/network-instance=default/protocols/protocol=STATIC,static/static-routes" \
  -H "accept: */*" \
  -H "Content-Type: application/yang-data+json" \
  -d '{
    "static": [
      {
        "prefix": "192.168.1.0/24",
        "config": {
          "prefix": "192.168.1.0/24"
        },
        "next-hops": {
          "next-hop": [
            {
              "index": "1",
              "config": {
                "index": "1"
              },
              "interface-ref": {
                "config": {
                  "interface": "Ethernet0"
                }
              }
            }
          ]
        }
      }
    ]
  }'
```

**Example: POST static route with VRF leak (extension)**
```bash
curl -X POST -k "https://192.168.1.1/restconf/data/openconfig-network-instance:network-instances/network-instance=Vrf1/protocols/protocol=STATIC,static/static-routes" \
  -H "accept: */*" \
  -H "Content-Type: application/yang-data+json" \
  -d '{
    "static": [
      {
        "prefix": "10.2.0.0/16",
        "config": {
          "prefix": "10.2.0.0/16"
        },
        "next-hops": {
          "next-hop": [
            {
              "index": "1",
              "config": {
                "index": "1",
                "next-hop": "10.0.0.1",
                "openconfig-local-routing-ext:nh-network-instance": "Vrf2"
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

**Example: PUT next-hop configuration**
```bash
curl -X PUT -k "https://192.168.1.1/restconf/data/openconfig-network-instance:network-instances/network-instance=default/protocols/protocol=STATIC,static/static-routes/static=10.1.1.0%2F24/next-hops/next-hop=1/config" \
  -H "accept: */*" \
  -H "Content-Type: application/yang-data+json" \
  -d '{
    "config": {
      "index": "1",
      "next-hop": "10.0.0.2"
    }
  }'
```

#### 3.3.3.4 PATCH
Used to modify specific fields without replacing entire configuration. Supported at all levels.

**Example: PATCH BFD enablement**
```bash
curl -X PATCH -k "https://192.168.1.1/restconf/data/openconfig-network-instance:network-instances/network-instance=default/protocols/protocol=STATIC,static/static-routes/static=10.1.1.0%2F24/next-hops/next-hop=1/enable-bfd/config/enabled" \
  -H "accept: */*" \
  -H "Content-Type: application/yang-data+json" \
  -d '{"enabled": true}'
```

**Example: PATCH next-hop address**
```bash
curl -X PATCH -k "https://192.168.1.1/restconf/data/openconfig-network-instance:network-instances/network-instance=default/protocols/protocol=STATIC,static/static-routes/static=10.1.1.0%2F24/next-hops/next-hop=1/config/next-hop" \
  -H "accept: */*" \
  -H "Content-Type: application/yang-data+json" \
  -d '{"next-hop": "10.0.0.3"}'
```

#### 3.3.3.5 DELETE
Supported at all levels.

**Example: DELETE specific next-hop**
```bash
curl -X DELETE -k "https://192.168.1.1/restconf/data/openconfig-network-instance:network-instances/network-instance=default/protocols/protocol=STATIC,static/static-routes/static=10.1.1.0%2F24/next-hops/next-hop=1" \
  -H "accept: */*"
```

**Example: DELETE entire static route**
```bash
curl -X DELETE -k "https://192.168.1.1/restconf/data/openconfig-network-instance:network-instances/network-instance=default/protocols/protocol=STATIC,static/static-routes/static=10.1.1.0%2F24" \
  -H "accept: */*"
```

### 3.3.4 gNMI Support

#### 3.3.4.1 GET

**Example: GET static routes**
```bash
gnmi_get -insecure -logtostderr -username admin -password password \
  -target_addr localhost:8080 \
  -xpath /openconfig-network-instance:network-instances/network-instance[name=default]/protocols/protocol[identifier=STATIC][name=static]/static-routes
```

#### 3.3.4.2 SET

**Example: SET static route with next-hop**
```bash
# Create static_route.json:
{
  "openconfig-local-routing:static": [
    {
      "prefix": "10.1.1.0/24",
      "config": {
        "prefix": "10.1.1.0/24"
      },
      "next-hops": {
        "next-hop": [
          {
            "index": "1",
            "config": {
              "index": "1",
              "next-hop": "10.0.0.1"
            }
          }
        ]
      }
    }
  ]
}

gnmi_set -insecure -logtostderr -username admin -password password \
  -target_addr localhost:8080 \
  -xpath_target OC-YANG \
  -update /openconfig-network-instance:network-instances/network-instance[name=default]/protocols/protocol[identifier=STATIC][name=static]/static-routes/static:@/tmp/static_route.json
```

#### 3.3.4.3 DELETE

**Example: DELETE static route**
```bash
gnmi_set -insecure -logtostderr -username admin -password password \
  -target_addr localhost:8080 \
  -xpath_target OC-YANG \
  -delete "/openconfig-network-instance:network-instances/network-instance[name=default]/protocols/protocol[identifier=STATIC][name=static]/static-routes/static[prefix=10.1.1.0/24]"
```

#### 3.3.4.4 SUBSCRIBE

gNMI subscription is supported for monitoring configuration changes on static route parameters.

# 4 Error Handling

The implementation handles various error scenarios and returns appropriate error responses:

- **Network Instance Not Found**: Attempting to configure static route in a non-existent network instance will return an error.
- **Invalid Prefix Format**: Providing an invalid IP prefix format will return an error.
- **Invalid Next-hop**: Providing an invalid next-hop IP address will return an error.
- **Interface Not Found**: Attempting to use a non-existent interface for next-hop will return an error.
- **Invalid Protocol**: Attempting to configure static routes under non-STATIC protocol will return an error.
- **Duplicate Route**: Attempting to create a duplicate static route entry will return an error.

# 5 Unit Test Cases

Comprehensive test cases are available in the transformer test suite.

## 5.1 Functional Test Cases

### 5.1.1 Static Route Configuration Tests

- POST static route with IPv4 prefix
- POST static route with IPv6 prefix
- GET all static routes
- GET specific static route
- DELETE static route

### 5.1.2 Next-hop Configuration Tests

- POST next-hop with IP address
- POST next-hop with interface
- POST next-hop with both IP and interface
- POST multiple next-hops for same prefix
- PATCH next-hop IP address
- DELETE specific next-hop
- GET all next-hops for a route

### 5.1.3 BFD Configuration Tests

- POST next-hop with BFD enabled
- PATCH BFD enablement
- GET BFD state
- DELETE next-hop with BFD enabled

### 5.1.4 VRF Configuration Tests

- POST static route in non-default VRF
- POST next-hop with nh-network-instance (VRF leak)
- GET static routes from specific VRF
- DELETE static route from VRF

### 5.1.5 Integration Tests

- Configure static route with multiple next-hops
- Configure static routes in multiple VRFs
- Configure static route with BFD and interface
- Modify existing static route next-hops
- GET entire static-routes container

## 5.2 Negative Test Cases

- POST static route with invalid prefix format
- POST next-hop with invalid IP address
- POST next-hop with non-existent interface
- POST static route in non-existent VRF
- POST next-hop with non-existent nh-network-instance
- DELETE non-existing static route
- PATCH next-hop with invalid index
- POST duplicate next-hop with same index
- POST static route under non-STATIC protocol

# 6 References

1. [OpenConfig Network Instance YANG Model](https://github.com/openconfig/public/tree/master/release/models/network-instance)
2. [OpenConfig Local Routing YANG Model](https://github.com/openconfig/public/tree/master/release/models/local-routing)
3. [SONiC Management Framework HLD](https://github.com/project-arlo/SONiC/blob/master/doc/mgmt/Management%20Framework.md)
4. [SONiC Static Route Configuration](https://github.com/sonic-net/SONiC/wiki/Static-Route)
5. [RFC 5880 - Bidirectional Forwarding Detection (BFD)](https://tools.ietf.org/html/rfc5880)
