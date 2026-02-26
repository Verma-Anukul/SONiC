# OpenConfig support for Network Instance

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
This document provides general information about the OpenConfig configuration of Network Instance in SONiC.

# Scope
- This document describes the high level design of Network Instance configuration using OpenConfig models via REST & gNMI.
- This does not cover the SONiC KLISH CLI.
- This covers Network Instance type, FDB (anycast gateway MAC and MAC table entries), interfaces, VLANs, and protocols configuration.
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
        +--rw fdb
        |  +--rw config
        |  |  +--rw anycast-gateway-mac?   oc-yang:mac-address
        |  +--ro state
        |  |  +--ro anycast-gateway-mac?   oc-yang:mac-address
        |  +--rw mac-table
        |     +--rw entries
        |        +--rw entry* [mac-address vlan]
        |           +--rw mac-address    -> ../config/mac-address
        |           +--rw vlan           -> ../config/vlan
        |           +--rw config
        |           |  +--rw mac-address?   oc-yang:mac-address
        |           |  +--rw vlan?          -> ../../../../../../vlans/vlan/config/vlan-id
        |           +--ro state
        |           |  +--ro mac-address?   oc-yang:mac-address
        |           |  +--ro vlan?          -> ../../../../../../vlans/vlan/config/vlan-id
        |           |  +--ro entry-type?    enumeration
        |           +--rw interface
        |              +--rw interface-ref
        |                 +--rw config
        |                 |  +--rw interface?      -> /oc-if:interfaces/interface/name
        |                 |  +--rw subinterface?   -> /oc-if:interfaces/interface[oc-if:name=current()/../interface]/subinterfaces/subinterface/index
        |                 +--ro state
        |                    +--ro interface?      -> /oc-if:interfaces/interface/name
        |                    +--ro subinterface?   -> /oc-if:interfaces/interface[oc-if:name=current()/../interface]/subinterfaces/subinterface/index
        +--rw interfaces
        |  +--rw interface* [id]
        |     +--rw id        -> ../config/id
        |     +--rw config
        |     |  +--rw id?             oc-if:interface-id
        |     |  +--rw interface?      -> /oc-if:interfaces/interface/name
        |     |  +--rw subinterface?   -> /oc-if:interfaces/interface[oc-if:name=current()/../interface]/subinterfaces/subinterface/index
        |     +--ro state
        |        +--ro id?             oc-if:interface-id
        |        +--ro interface?      -> /oc-if:interfaces/interface/name
        |        +--ro subinterface?   -> /oc-if:interfaces/interface[oc-if:name=current()/../interface]/subinterfaces/subinterface/index
        +--rw vlans
        |  +--rw vlan* [vlan-id]
        |     +--rw vlan-id    -> ../config/vlan-id
        |     +--rw config
        |     |  +--rw vlan-id?                                          oc-vlan-types:vlan-id
        |     |  +--rw name?                                             string
        |     |  +--rw status?                                           enumeration
        |     |  +--rw oc-network-instance-ext:static-anycast-gateway?   boolean
        |     +--ro state
        |     |  +--ro vlan-id?                                          oc-vlan-types:vlan-id
        |     |  +--ro name?                                             string
        |     |  +--ro status?                                           enumeration
        |     |  +--ro oc-network-instance-ext:static-anycast-gateway?   boolean
        |     +--rw members
        |        +--ro member* []
        |           +--ro state
        |              +--ro interface?   base-interface-ref
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
```

# Definition/Abbreviation
### Table 1: Abbreviations
| **Term**                 | **Definition**                         |
|--------------------------|-------------------------------------|
| YANG                     | Yet Another Next Generation: modular language representing data structures in an XML tree format        |
| REST                     | REpresentative State Transfer |
| gNMI                     | gRPC Network Management Interface: used to retrieve or manipulate the state of a device via telemetry or configuration data         |
| VRF                      | Virtual Routing and Forwarding   |
| FDB                      | Forwarding Database   |
| MAC                      | Media Access Control   |

# 1 Feature Overview
## 1.1 Requirements
### 1.1.1 Functional Requirements
1. Provide support for OpenConfig YANG models for Network Instance configuration.
2. Configure/Set, GET, and Delete Network Instance parameters.
3. Support configuration of network instance type (default, L3VRF, etc.).
4. Support configuration of anycast gateway MAC address.
5. Support retrieval of MAC table entries (read-only operational state from ASIC_DB).
6. Support binding interfaces to network instances.
7. Support VLAN configuration within network instances.
8. Support protocol configuration within network instances.
9. Support static anycast gateway configuration for VLANs (extension).
10. Map OpenConfig network-instance model to SONiC CONFIG_DB tables.

### 1.1.2 Configuration and Management Requirements
The Network Instance configurations can be done via REST and gNMI. The implementation will return an error if a configuration is not allowed.

## 1.2 Design Overview
### 1.2.1 Basic Approach
SONiC already supports Network Instance configurations via CLI. This feature adds support for OpenConfig based YANG models using transformer based implementation in the Management Framework.

The implementation provides mapping between:
- OpenConfig network-instance config parameters → SONiC VRF and VLAN tables
- OpenConfig anycast gateway configuration → SONiC SAG (Static Anycast Gateway) table
- OpenConfig MAC table entries (read-only) → ASIC_DB operational state
- OpenConfig VLAN configuration → SONiC VLAN tables
- OpenConfig interface bindings → SONiC VLAN_MEMBER and INTERFACE tables

### 1.2.2 Container
The code changes for this feature are part of *mgmt-framework* container which includes the REST server and *gnmi* container for gNMI support in *sonic-mgmt-common* repository.

# 2 Functionality
## 2.1 Target Deployment Use Cases
1. REST client through which the user can perform PATCH, DELETE, POST, PUT, and GET operations on Network Instance configuration paths.
2. gNMI client with support for capabilities, get, set, and subscribe operations based on the supported YANG models.

# 3 Design
## 3.1 Overview
This HLD design is in line with the [Management Framework HLD](https://github.com/project-arlo/SONiC/blob/master/doc/mgmt/Management%20Framework.md)

The implementation uses transformer functions in `translib/transformer/xfmr_network_instance.go` to map between OpenConfig and SONiC data models.

### 3.1.1 Mapping Details

#### Table 2: OpenConfig YANG to SONiC YANG Mapping

| OpenConfig YANG Node | SONiC YANG File | DB Name | Table:Field |
|---------------------|-----------------|---------|-------------|
| **network-instance** | | | |
| config/name | sonic-vrf.yang | CONFIG_DB | VRF:vrf_name |
| config/type | sonic-vrf.yang | CONFIG_DB | VRF:type |
| **fdb** | | | |
| config/anycast-gateway-mac | sonic-static-anycast-gateway.yang | CONFIG_DB | SAG\|GLOBAL:gateway_mac |
| **fdb/mac-table/entries/entry** | | | |
| state/mac-address | N/A | ASIC_DB | ASIC_STATE:SAI_OBJECT_TYPE_FDB_ENTRY (read-only) |
| state/vlan | N/A | ASIC_DB | ASIC_STATE:SAI_OBJECT_TYPE_FDB_ENTRY (read-only) |
| state/entry-type | N/A | ASIC_DB | ASIC_STATE:SAI_OBJECT_TYPE_FDB_ENTRY attr SAI_FDB_ENTRY_ATTR_TYPE (read-only) |
| interface/interface-ref/state/interface | N/A | ASIC_DB + COUNTERS_DB | ASIC_STATE:SAI_OBJECT_TYPE_FDB_ENTRY + COUNTERS:COUNTERS_PORT_NAME_MAP (read-only) |
| **interfaces/interface** | | | |
| config/id | sonic-interface.yang | CONFIG_DB | INTERFACE\|VLAN_SUB_INTERFACE:interface (VRF binding via vrf_name field) |
| config/interface | sonic-interface.yang | CONFIG_DB | INTERFACE\|VLAN_SUB_INTERFACE:interface (VRF binding via vrf_name field) |
| **vlans/vlan** | | | |
| config/vlan-id | sonic-vlan.yang | CONFIG_DB | VLAN:vlanid |
| config/name | sonic-vlan.yang | CONFIG_DB | VLAN:name |
| config/status | sonic-vlan.yang | CONFIG_DB | VLAN:admin_status |
| config/static-anycast-gateway (ext) | sonic-vlan.yang | CONFIG_DB | VLAN_INTERFACE\|VlanX:static_anycast_gateway |
| members/member/state/interface | sonic-vlan.yang | CONFIG_DB | VLAN_MEMBER:`<key>` |
| **protocols/protocol** | | | |
| config/identifier | N/A | N/A | Not stored in VRF table - protocol-specific |
| config/name | N/A | N/A | Not stored in VRF table - protocol-specific |

**Notes:**
- **Bold** entries indicate major feature categories/containers
- State nodes mirror their corresponding config nodes and are read-only
- **FDB (MAC table) key formats:**
  - CONFIG_DB FDB table: `"Vlan<id>:<mac-address>"` (for static MAC entries)
  - ASIC_DB JSON key for OpenConfig GET: `{"bvid":"<oid>","mac":"<mac>","switch_id":"<oid>"}`
  - ASIC_DB table: `ASIC_STATE:SAI_OBJECT_TYPE_FDB_ENTRY`
- Key format for VLAN_MEMBER: `"VlanX|interface-name"`
- Key format for VLAN_INTERFACE: `"VlanX"` or `"VlanX|<ip-prefix>"` (for static anycast gateway)
- **Interface to VRF binding:** Uses `INTERFACE`, `VLAN_SUB_INTERFACE`, `PORTCHANNEL_INTERFACE`, etc. with `vrf_name` field (not VLAN_MEMBER)
- Network instance type maps to VRF configuration in SONiC
- Anycast gateway MAC is stored in SAG (Static Anycast Gateway) table with key `GLOBAL`
- MAC table state data is retrieved from ASIC_DB `SAI_OBJECT_TYPE_FDB_ENTRY` for read-only attributes
- **MAC table interface resolution:** Requires mapping from ASIC_DB bridge port OID to interface name using COUNTERS_DB `COUNTERS_PORT_NAME_MAP`
- **MAC table entries are read-only operational state - cannot be configured via OpenConfig**
- **VLAN static anycast gateway:** Stored in `VLAN_INTERFACE` table with key `VlanX` and field `static_anycast_gateway` (true/false)
- Protocol configuration (BGP, static routes, etc.) is not stored in VRF table - each protocol has its own tables

## 3.2 DB Changes

### 3.2.1 CONFIG DB

The implementation uses existing SONiC YANG schemas for VRF, VLAN, and FDB configuration.

### 3.2.2 APPL_DB
MAC table entries are retrieved from APPL_DB FDB_TABLE for learning status, but static configuration entries come from ASIC_DB for state data.

### 3.2.3 STATE DB
There are no changes to STATE DB schema definition for this feature. MAC table state information is retrieved from ASIC_DB.

### 3.2.4 ASIC DB
There are no changes to ASIC DB schema definition.

### 3.2.5 COUNTER DB
There are no changes to COUNTER DB schema definition.

## 3.3 User Interface
### 3.3.1 Data Models
The implementation uses standard OpenConfig YANG models:
- **openconfig-network-instance.yang**: Main network instance model
- **openconfig-vlan.yang**: VLAN definitions
- **openconfig-interfaces.yang**: Interface references

### 3.3.2 OpenConfig Extensions

The following extensions are added to the standard OpenConfig network-instance model in **openconfig-network-instance-ext.yang**:

#### VLAN Static Anycast Gateway Extension
```yang
augment /oc-ni:network-instances/oc-ni:network-instance/oc-ni:vlans/oc-ni:vlan/oc-ni:config {
  leaf static-anycast-gateway {
    type boolean;
    description "Enable static anycast gateway for this VLAN";
  }
}
```

**Purpose**: Enables static anycast gateway functionality per VLAN, allowing the VLAN to use the global anycast MAC address for gateway services.

**Usage**: Set to `true` to enable anycast gateway for a specific VLAN.

### 3.3.3 REST API Support

#### 3.3.3.1 GET
Supported at all levels (container, list, and leaf).

**Example: GET network instance config**
```bash
curl -X GET -k "https://192.168.1.1/restconf/data/openconfig-network-instance:network-instances/network-instance=default" -H "accept: application/yang-data+json"
```

Response:
```json
{
  "openconfig-network-instance:network-instance": [
    {
      "name": "default",
      "config": {
        "name": "default",
        "type": "DEFAULT_INSTANCE"
      }
    }
  ]
}
```

**Example: GET anycast gateway MAC**
```bash
curl -X GET -k "https://192.168.1.1/restconf/data/openconfig-network-instance:network-instances/network-instance=default/fdb/config/anycast-gateway-mac" -H "accept: application/yang-data+json"
```

#### 3.3.3.2 POST
Used to create new configuration. Supported at container and leaf levels.

**Example: POST a new network instance**
```bash
curl -X POST -k "https://192.168.1.1/restconf/data/openconfig-network-instance:network-instances" \
  -H "accept: */*" \
  -H "Content-Type: application/yang-data+json" \
  -d '{
    "network-instance": [
      {
        "name": "Vrf1",
        "config": {
          "name": "Vrf1",
          "type": "L3VRF"
        }
      }
    ]
  }'
```

**Note:** MAC table entries are read-only operational state from ASIC_DB and cannot be created via POST.

#### 3.3.3.3 PUT
Used to replace entire configuration. Supported at container and leaf levels.

**Example: PUT anycast gateway MAC**
```bash
curl -X PUT -k "https://192.168.1.1/restconf/data/openconfig-network-instance:network-instances/network-instance=default/fdb/config/anycast-gateway-mac" \
  -H "accept: */*" \
  -H "Content-Type: application/yang-data+json" \
  -d '{"anycast-gateway-mac": "00:11:22:33:44:55"}'
```

#### 3.3.3.4 PATCH
Used to modify specific fields without replacing entire configuration. Supported at all levels.

**Example: PATCH VLAN status**
```bash
curl -X PATCH -k "https://192.168.1.1/restconf/data/openconfig-network-instance:network-instances/network-instance=default/vlans/vlan=100/config/status" \
  -H "accept: */*" \
  -H "Content-Type: application/yang-data+json" \
  -d '{"status": "ACTIVE"}'
```

**Example: PATCH static anycast gateway (extension)**
```bash
curl -X PATCH -k "https://192.168.1.1/restconf/data/openconfig-network-instance:network-instances/network-instance=default/vlans/vlan=100/config/openconfig-network-instance-ext:static-anycast-gateway" \
  -H "accept: */*" \
  -H "Content-Type: application/yang-data+json" \
  -d '{"openconfig-network-instance-ext:static-anycast-gateway": true}'
```

#### 3.3.3.5 DELETE
Supported at all levels.

**Example: DELETE network instance**
```bash
curl -X DELETE -k "https://192.168.1.1/restconf/data/openconfig-network-instance:network-instances/network-instance=Vrf1" \
  -H "accept: */*"
```

**Note:** MAC table entries cannot be deleted via OpenConfig - they are read-only operational state.

### 3.3.4 gNMI Support

#### 3.3.4.1 GET

**Example: GET network instance**
```bash
gnmi_get -insecure -logtostderr -username admin -password password \
  -target_addr localhost:8080 \
  -xpath /openconfig-network-instance:network-instances/network-instance[name=default]
```

#### 3.3.4.2 SET

**Example: SET anycast gateway MAC**
```bash
gnmi_set -insecure -logtostderr -username admin -password password \
  -target_addr localhost:8080 \
  -xpath_target OC-YANG \
  -update "/openconfig-network-instance:network-instances/network-instance[name=default]/fdb/config/anycast-gateway-mac:00:11:22:33:44:55"
```

#### 3.3.4.3 DELETE

**Example: DELETE network instance**
```bash
gnmi_set -insecure -logtostderr -username admin -password password \
  -target_addr localhost:8080 \
  -xpath_target OC-YANG \
  -delete /openconfig-network-instance:network-instances/network-instance[name=Vrf1]
```

#### 3.3.4.4 SUBSCRIBE

gNMI subscription is supported for monitoring configuration changes on network instance parameters.

# 4 Error Handling

The implementation handles various error scenarios and returns appropriate error responses:

- **Network Instance Not Found**: Attempting to configure on a non-existent network instance will return an error.
- **Invalid MAC Address**: Providing an invalid MAC address format will return an error.
- **VLAN Not Found**: Attempting to configure on a non-existent VLAN will return an error.
- **Interface Not Found**: Attempting to bind a non-existent interface will return an error.
- **Duplicate Entry**: Attempting to create a duplicate MAC table entry will return an error.
- **Protocol Configuration**: Protocol-level configuration (identifier, name) at network-instance/protocols/protocol level is not supported. Each protocol (BGP, static routes, etc.) has its own configuration paths and tables.

# 5 Unit Test Cases

Comprehensive test cases are available in the transformer test suite.

## 5.1 Functional Test Cases

### 5.1.1 Network Instance Configuration Tests

- POST network instance with type=L3VRF
- GET network instance config
- PATCH network instance type
- DELETE network instance
- GET all network instances

### 5.1.2 FDB Configuration Tests

- POST anycast gateway MAC address
- GET anycast gateway MAC
- PATCH anycast gateway MAC
- DELETE anycast gateway MAC

### 5.1.3 MAC Table Tests

- GET all MAC table entries (read-only)
- GET specific MAC table entry (read-only)
- GET MAC table entry state (entry-type, interface)
- Verify MAC entries cannot be created/deleted via OpenConfig

### 5.1.4 VLAN Configuration Tests

- POST VLAN in network instance
- GET VLAN config
- PATCH VLAN status
- PATCH static anycast gateway (extension)
- GET VLAN members
- DELETE VLAN

### 5.1.5 Interface Binding Tests

- POST interface to network instance
- GET interface bindings
- DELETE interface from network instance

### 5.1.6 Integration Tests

- Configure network instance with VLANs and interfaces
- Configure multiple MAC table entries
- Configure anycast gateway with multiple VLANs
- GET entire network instance container

## 5.2 Negative Test Cases

- Configure on non-existent network instance
- POST duplicate network instance
- POST invalid MAC address format
- POST MAC entry with non-existent VLAN
- POST MAC entry with non-existent interface
- DELETE non-existing MAC entry
- PATCH VLAN in wrong network instance
- POST interface binding with invalid interface name

# 6 References

1. [OpenConfig Network Instance YANG Model](https://github.com/openconfig/public/tree/master/release/models/network-instance)
2. [OpenConfig VLAN YANG Model](https://github.com/openconfig/public/tree/master/release/models/vlan)
3. [SONiC Management Framework HLD](https://github.com/project-arlo/SONiC/blob/master/doc/mgmt/Management%20Framework.md)
4. [SONiC VRF HLD](https://github.com/sonic-net/SONiC/blob/master/doc/vrf/sonic-vrf-hld.md)
