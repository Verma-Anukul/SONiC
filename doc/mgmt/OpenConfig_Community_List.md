# OpenConfig support for Community List

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
      * [3.2.2 APP DB](#322-app-db)
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
This document provides general information about the OpenConfig configuration of Community Lists in SONiC.

# Scope
- This document describes the high level design of Community List configuration using OpenConfig models via REST & gNMI.
- This does not cover the SONiC KLISH CLI.
- This covers BGP standard community lists and extended community lists.
- This includes OpenConfig extensions for community list features.
- Supported attributes in OpenConfig YANG tree:

```yang
module: openconfig-routing-policy
  +--rw routing-policy
     +--rw defined-sets
        +--rw oc-bgp-pol:bgp-defined-sets
           +--rw oc-bgp-pol:community-sets
           |  +--rw oc-bgp-pol:community-set* [community-set-name]
           |     +--rw oc-bgp-pol:community-set-name    -> ../config/community-set-name
           |     +--rw oc-bgp-pol:config
           |     |  +--rw oc-bgp-pol:community-set-name    string
           |     |  +--rw oc-bgp-pol:community-member*     union
           |     |  +--rw oc-rp-ext:action?                action-type
           |     +--ro oc-bgp-pol:state
           |        +--ro oc-bgp-pol:community-set-name    string
           |        +--ro oc-bgp-pol:community-member*     union
           |        +--ro oc-rp-ext:action?                action-type
           +--rw oc-bgp-pol:ext-community-sets
              +--rw oc-bgp-pol:ext-community-set* [ext-community-set-name]
                 +--rw oc-bgp-pol:ext-community-set-name    -> ../config/ext-community-set-name
                 +--rw oc-bgp-pol:config
                 |  +--rw oc-bgp-pol:ext-community-set-name?   string
                 |  +--rw oc-bgp-pol:ext-community-member*     union
                 |  +--rw oc-rp-ext:action?                    action-type
                 +--ro oc-bgp-pol:state
                    +--ro oc-bgp-pol:ext-community-set-name?   string
                    +--ro oc-bgp-pol:ext-community-member*     union
                    +--ro oc-rp-ext:action?                    action-type
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
| RT                       | Route Target   |
| RO                       | Route Origin   |

# 1 Feature Overview
## 1.1 Requirements
### 1.1.1 Functional Requirements
1. Provide support for OpenConfig YANG models for Community List configuration.
2. Configure/Set, GET, and Delete standard community lists.
3. Configure/Set, GET, and Delete extended community lists.
4. Support multiple community members per community set.
5. Support community format: "AS:NN" (e.g., "65000:100").
6. Support well-known communities (NO_EXPORT, NO_ADVERTISE, etc.).
7. Support regular expression matching for communities.
8. Support extended community types (RT, RO).
9. Support permit/deny actions for community sets (extension).
10. Map OpenConfig community-sets model to SONiC COMMUNITY_SET table.

### 1.1.2 Configuration and Management Requirements
The Community List configurations can be done via REST and gNMI. The implementation will return an error if a configuration is not allowed.

**Important Notes:**
- Community lists are used in routing policies (route maps) to match BGP communities.
- Standard communities use format "AS:NN" or well-known names.
- Extended communities include Route Target (RT) and Route Origin (RO).

## 1.2 Design Overview
### 1.2.1 Basic Approach
SONiC uses FRR (Free Range Routing) for BGP and routing policy implementation. This feature adds support for OpenConfig based YANG models using transformer based implementation in the Management Framework.

The implementation provides mapping between:
- OpenConfig community-sets config → SONiC COMMUNITY_SET table
- OpenConfig ext-community-sets config → SONiC EXTENDED_COMMUNITY_SET table (separate table for extended communities)
- OpenConfig extensions → SONiC-specific community list features

### 1.2.2 Container
The code changes for this feature are part of *mgmt-framework* container which includes the REST server and *gnmi* container for gNMI support in *sonic-mgmt-common* repository.

# 2 Functionality
## 2.1 Target Deployment Use Cases
1. REST client through which the user can perform PATCH, DELETE, POST, PUT, and GET operations on Community List configuration paths.
2. gNMI client with support for capabilities, get, set, and subscribe operations based on the supported YANG models.
3. Community lists used as match conditions in BGP routing policies (route maps).

# 3 Design
## 3.1 Overview
This HLD design is in line with the [Management Framework HLD](https://github.com/project-arlo/SONiC/blob/master/doc/mgmt/Management%20Framework.md)

The implementation uses transformer functions in `translib/transformer/xfmr_community_list.go` to map between OpenConfig and SONiC data models.

### 3.1.1 Mapping Details

#### Table 2: OpenConfig YANG to SONiC YANG Mapping

| OpenConfig YANG Node | SONiC YANG File | DB Name | Table:Field |
|---------------------|-----------------|---------|-------------|
| **bgp-defined-sets/community-sets/community-set** | | | |
| config/community-set-name | sonic-routing-policy.yang | CONFIG_DB | COMMUNITY_SET:`<name>` |
| config/community-member | sonic-routing-policy.yang | CONFIG_DB | COMMUNITY_SET:community_member@ |
| config/match-set-options | sonic-routing-policy.yang | CONFIG_DB | COMMUNITY_SET:match_action |
| config/action (extension) | sonic-routing-policy.yang | CONFIG_DB | COMMUNITY_SET:action |
| **bgp-defined-sets/ext-community-sets/ext-community-set** | | | |
| config/ext-community-set-name | sonic-routing-policy.yang | CONFIG_DB | EXTENDED_COMMUNITY_SET:`<name>` |
| config/ext-community-member | sonic-routing-policy.yang | CONFIG_DB | EXTENDED_COMMUNITY_SET:community_member@ |
| config/match-set-options | sonic-routing-policy.yang | CONFIG_DB | EXTENDED_COMMUNITY_SET:match_action |
| config/action (extension) | sonic-routing-policy.yang | CONFIG_DB | EXTENDED_COMMUNITY_SET:action |

**Notes:**
- **Bold** entries indicate major feature categories/containers
- State nodes mirror their corresponding config nodes and are read-only
- Standard community sets use COMMUNITY_SET table
- Extended community sets use EXTENDED_COMMUNITY_SET table (separate table)
- Field **set_type** is used internally by transformers to distinguish between STANDARD and EXPANDED (regex) community sets; not directly mapped from OpenConfig
- Field **action** specifies permit/deny behavior (permit or deny) via SONiC extension; not present in standard OpenConfig annotations
- Field **community_member@** stores community members for both COMMUNITY_SET and EXTENDED_COMMUNITY_SET (@ indicates array/list type in Redis)
- Extended community format: "rt:AS:NN" or "soo:AS:NN"
- Match action values: ANY, ALL

## 3.2 DB Changes

### 3.2.1 CONFIG DB

The **sonic-routing-policy.yang** schema is used for community list configuration.

**CONFIG_DB Examples:**

**COMMUNITY_SET Table (Standard):**
```
COMMUNITY_SET|standard-comm-list
  "set_type": "STANDARD"
  "action": "permit"
  "match_action": "ANY"
  "community_member@": "65000:100,65000:200,NO_EXPORT"
```

**COMMUNITY_SET Table (Expanded - Regex):**
```
COMMUNITY_SET|expanded-comm-list
  "set_type": "EXPANDED"
  "action": "permit"
  "match_action": "ANY"
  "community_member@": "^65[0-9]+:.*"
```

**EXTENDED_COMMUNITY_SET Table:**
```
EXTENDED_COMMUNITY_SET|ext-comm-list
  "action": "permit"
  "match_action": "ANY"
  "ext_community_member@": "rt:65000:100,soo:65000:200"
```

**Note:** 
- Standard and expanded community sets are stored in COMMUNITY_SET table with `set_type` field distinguishing them
- Extended community sets are stored in separate EXTENDED_COMMUNITY_SET table
- The `action` field specifies permit/deny behavior

**FRR Configuration:**

Changes are made in FRR configuration to support community lists. The frrcfgd daemon subscribes to COMMUNITY_SET and EXTENDED_COMMUNITY_SET tables and configures corresponding FRR bgp community-list and bgp extcommunity-list commands.

### 3.2.2 APP DB
There are no changes to APP DB schema definition for this feature.

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
- **openconfig-bgp-types.yang**: BGP community type definitions

### 3.3.2 OpenConfig Extensions

The following extension is added to the standard OpenConfig routing-policy model in **openconfig-routing-policy-ext.yang**:

#### Community Set Action
```yang
augment /oc-rpol:routing-policy/oc-rpol:defined-sets/oc-bgp-pol:bgp-defined-sets/
        oc-bgp-pol:community-sets/oc-bgp-pol:community-set/oc-bgp-pol:config {
  leaf action {
    type enumeration {
      enum permit {
        description "Permit matching communities";
      }
      enum deny {
        description "Deny matching communities";
      }
    }
    description "Action for community list";
  }
}

augment /oc-rpol:routing-policy/oc-rpol:defined-sets/oc-bgp-pol:bgp-defined-sets/
        oc-bgp-pol:ext-community-sets/oc-bgp-pol:ext-community-set/oc-bgp-pol:config {
  leaf action {
    type enumeration {
      enum permit {
        description "Permit matching extended communities";
      }
      enum deny {
        description "Deny matching extended communities";
      }
    }
    description "Action for extended community list";
  }
}
```

**Purpose**: Defines whether matching communities should be permitted or denied.

**Usage**: Set to "permit" to allow matching communities, "deny" to block them. This is used when the community list is referenced in a route map.

### 3.3.3 REST API Support

#### 3.3.3.1 GET
Supported at all levels (container, list, and leaf).

**Example: GET all community sets**
```bash
curl -X GET -k "https://192.168.1.1/restconf/data/openconfig-routing-policy:routing-policy/defined-sets/openconfig-bgp-policy:bgp-defined-sets/community-sets" -H "accept: application/yang-data+json"
```

Response:
```json
{
  "openconfig-bgp-policy:community-sets": {
    "community-set": [
      {
        "community-set-name": "standard-comm-list",
        "config": {
          "community-set-name": "standard-comm-list",
          "community-member": ["65000:100", "65000:200", "NO_EXPORT"]
        }
      }
    ]
  }
}
```

**Example: GET extended community sets**
```bash
curl -X GET -k "https://192.168.1.1/restconf/data/openconfig-routing-policy:routing-policy/defined-sets/openconfig-bgp-policy:bgp-defined-sets/ext-community-sets" -H "accept: application/yang-data+json"
```

#### 3.3.3.2 POST
Used to create new configuration. Supported at container and leaf levels.

**Example: POST a standard community set**
```bash
curl -X POST -k "https://192.168.1.1/restconf/data/openconfig-routing-policy:routing-policy/defined-sets/openconfig-bgp-policy:bgp-defined-sets/community-sets" \
  -H "accept: */*" \
  -H "Content-Type: application/yang-data+json" \
  -d '{
    "community-set": [
      {
        "community-set-name": "standard-comm-list",
        "config": {
          "community-set-name": "standard-comm-list",
          "community-member": ["65000:100", "65000:200", "NO_EXPORT"],
          "openconfig-routing-policy-ext:action": "permit"
        }
      }
    ]
  }'
```

**Example: POST an expanded community set (regex)**
```bash
curl -X POST -k "https://192.168.1.1/restconf/data/openconfig-routing-policy:routing-policy/defined-sets/openconfig-bgp-policy:bgp-defined-sets/community-sets" \
  -H "accept: */*" \
  -H "Content-Type: application/yang-data+json" \
  -d '{
    "community-set": [
      {
        "community-set-name": "expanded-comm-list",
        "config": {
          "community-set-name": "expanded-comm-list",
          "community-member": ["^65[0-9]+:.*"],
          "openconfig-routing-policy-ext:action": "permit"
        }
      }
    ]
  }'
```

**Example: POST an extended community set**
```bash
curl -X POST -k "https://192.168.1.1/restconf/data/openconfig-routing-policy:routing-policy/defined-sets/openconfig-bgp-policy:bgp-defined-sets/ext-community-sets" \
  -H "accept: */*" \
  -H "Content-Type: application/yang-data+json" \
  -d '{
    "ext-community-set": [
      {
        "ext-community-set-name": "ext-comm-list",
        "config": {
          "ext-community-set-name": "ext-comm-list",
          "ext-community-member": ["rt:65000:100", "soo:65000:200"],
          "openconfig-routing-policy-ext:action": "permit"
        }
      }
    ]
  }'
```

**Example: POST with well-known communities**
```bash
curl -X POST -k "https://192.168.1.1/restconf/data/openconfig-routing-policy:routing-policy/defined-sets/openconfig-bgp-policy:bgp-defined-sets/community-sets" \
  -H "accept: */*" \
  -H "Content-Type: application/yang-data+json" \
  -d '{
    "community-set": [
      {
        "community-set-name": "well-known-comm",
        "config": {
          "community-set-name": "well-known-comm",
          "community-member": ["NO_EXPORT", "NO_ADVERTISE", "LOCAL_AS"],
          "openconfig-routing-policy-ext:action": "deny"
        }
      }
    ]
  }'
```

#### 3.3.3.3 PUT
Used to replace entire configuration. Supported at container and leaf levels.

**Example: PUT community set config**
```bash
curl -X PUT -k "https://192.168.1.1/restconf/data/openconfig-routing-policy:routing-policy/defined-sets/openconfig-bgp-policy:bgp-defined-sets/community-sets/community-set=standard-comm-list/config" \
  -H "accept: */*" \
  -H "Content-Type: application/yang-data+json" \
  -d '{
    "config": {
      "community-set-name": "standard-comm-list",
      "community-member": ["65000:100", "65000:300"],
      "openconfig-routing-policy-ext:action": "permit"
    }
  }'
```

#### 3.3.3.4 PATCH
Used to modify specific fields without replacing entire configuration. Supported at all levels.

**Example: PATCH community members**
```bash
curl -X PATCH -k "https://192.168.1.1/restconf/data/openconfig-routing-policy:routing-policy/defined-sets/openconfig-bgp-policy:bgp-defined-sets/community-sets/community-set=standard-comm-list/config/community-member" \
  -H "accept: */*" \
  -H "Content-Type: application/yang-data+json" \
  -d '{"community-member": ["65000:100", "65000:200", "65000:300"]}'
```

**Example: PATCH action (extension)**
```bash
curl -X PATCH -k "https://192.168.1.1/restconf/data/openconfig-routing-policy:routing-policy/defined-sets/openconfig-bgp-policy:bgp-defined-sets/community-sets/community-set=standard-comm-list/config/openconfig-routing-policy-ext:action" \
  -H "accept: */*" \
  -H "Content-Type: application/yang-data+json" \
  -d '{"openconfig-routing-policy-ext:action": "deny"}'
```

#### 3.3.3.5 DELETE
Supported at all levels.

**Example: DELETE community set**
```bash
curl -X DELETE -k "https://192.168.1.1/restconf/data/openconfig-routing-policy:routing-policy/defined-sets/openconfig-bgp-policy:bgp-defined-sets/community-sets/community-set=standard-comm-list" \
  -H "accept: */*"
```

**Example: DELETE extended community set**
```bash
curl -X DELETE -k "https://192.168.1.1/restconf/data/openconfig-routing-policy:routing-policy/defined-sets/openconfig-bgp-policy:bgp-defined-sets/ext-community-sets/ext-community-set=ext-comm-list" \
  -H "accept: */*"
```

### 3.3.4 gNMI Support

#### 3.3.4.1 GET

**Example: GET community sets**
```bash
gnmi_get -insecure -logtostderr -username admin -password password \
  -target_addr localhost:8080 \
  -xpath /openconfig-routing-policy:routing-policy/defined-sets/openconfig-bgp-policy:bgp-defined-sets/community-sets
```

#### 3.3.4.2 SET

**Example: SET community set**
```bash
# Create community_set.json:
{
  "openconfig-bgp-policy:community-set": [
    {
      "community-set-name": "standard-comm-list",
      "config": {
        "community-set-name": "standard-comm-list",
        "community-member": ["65000:100", "65000:200"]
      }
    }
  ]
}

gnmi_set -insecure -logtostderr -username admin -password password \
  -target_addr localhost:8080 \
  -xpath_target OC-YANG \
  -update /openconfig-routing-policy:routing-policy/defined-sets/openconfig-bgp-policy:bgp-defined-sets/community-sets/community-set:@/tmp/community_set.json
```

#### 3.3.4.3 DELETE

**Example: DELETE community set**
```bash
gnmi_set -insecure -logtostderr -username admin -password password \
  -target_addr localhost:8080 \
  -xpath_target OC-YANG \
  -delete "/openconfig-routing-policy:routing-policy/defined-sets/openconfig-bgp-policy:bgp-defined-sets/community-sets/community-set[community-set-name=standard-comm-list]"
```

#### 3.3.4.4 SUBSCRIBE

gNMI subscription is supported for monitoring configuration changes on community list parameters.

# 4 Error Handling

The implementation handles various error scenarios and returns appropriate error responses:

- **Community Set Not Found**: Attempting to configure on a non-existent community set will return an error.
- **Invalid Community Format**: Providing an invalid community format will return an error.
- **Invalid Extended Community Format**: Providing an invalid extended community format will return an error.
- **Invalid Regex**: Providing an invalid regular expression in expanded community list will return an error.
- **Duplicate Community Set**: Attempting to create a duplicate community set will return an error.
- **Invalid Action**: Providing an invalid action value will return an error.

# 5 Unit Test Cases

Comprehensive test cases are available in `translib/transformer/xfmr_community_list_test.go`.

## 5.1 Functional Test Cases

### 5.1.1 Standard Community Set Tests

- POST standard community set with AS:NN format
- POST community set with well-known communities (NO_EXPORT, NO_ADVERTISE)
- POST community set with multiple members
- POST community set with permit action
- POST community set with deny action
- GET all community sets
- GET specific community set
- PATCH community members
- PATCH action
- DELETE community set

### 5.1.2 Expanded Community Set Tests

- POST expanded community set with regex pattern
- POST multiple regex patterns
- GET expanded community set
- PATCH regex pattern
- DELETE expanded community set

### 5.1.3 Extended Community Set Tests

- POST extended community set with RT (Route Target)
- POST extended community set with SOO (Site of Origin)
- POST extended community set with multiple members
- POST extended community set with mixed RT and SOO
- GET all extended community sets
- GET specific extended community set
- PATCH extended community members
- DELETE extended community set

### 5.1.4 Integration Tests

- Configure multiple standard community sets
- Configure multiple extended community sets
- Configure both standard and extended community sets
- Modify existing community set members
- GET entire bgp-defined-sets container

## 5.2 Negative Test Cases

- POST community set with invalid name
- POST community with invalid AS:NN format
- POST extended community with invalid RT format
- POST extended community with invalid SOO format
- POST community with invalid regex
- POST duplicate community set
- DELETE non-existing community set
- DELETE non-existing extended community set
- PATCH community with invalid action value
- POST well-known community with wrong spelling

# 6 References

1. [OpenConfig Routing Policy YANG Model](https://github.com/openconfig/public/tree/master/release/models/policy)
2. [OpenConfig BGP Policy YANG Model](https://github.com/openconfig/public/tree/master/release/models/bgp)
3. [SONiC Management Framework HLD](https://github.com/project-arlo/SONiC/blob/master/doc/mgmt/Management%20Framework.md)
4. [RFC 1997 - BGP Communities Attribute](https://tools.ietf.org/html/rfc1997)
5. [RFC 4360 - BGP Extended Communities Attribute](https://tools.ietf.org/html/rfc4360)
6. [RFC 8092 - BGP Large Communities Attribute](https://tools.ietf.org/html/rfc8092)
