# OpenConfig support for Prefix List

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
This document provides general information about the OpenConfig configuration of Prefix Lists in SONiC.

# Scope
- This document describes the high level design of Prefix List configuration using OpenConfig models via REST & gNMI.
- This does not cover the SONiC KLISH CLI.
- This covers IPv4 and IPv6 prefix list configuration with prefix matching and mask length ranges.
- This includes OpenConfig extensions for prefix list features.
- Supported attributes in OpenConfig YANG tree:

```yang
module: openconfig-routing-policy
  +--rw routing-policy
     +--rw defined-sets
        +--rw prefix-sets
           +--rw prefix-set* [name]
              +--rw name        -> ../config/name
              +--rw config
              |  +--rw name?                    string
              |  +--rw mode?                    enumeration
              |  +--rw oc-rp-ext:description?   string
              +--ro state
              |  +--ro name?                    string
              |  +--ro mode?                    enumeration
              |  +--ro oc-rp-ext:description?   string
              +--rw prefixes
                 +--rw prefix* [ip-prefix masklength-range]
                    +--rw ip-prefix           -> ../config/ip-prefix
                    +--rw masklength-range    -> ../config/masklength-range
                    +--rw config
                    |  +--rw ip-prefix                    oc-inet:ip-prefix
                    |  +--rw masklength-range?            string
                    |  +--rw oc-rp-ext:sequence-number?   uint32
                    |  +--rw oc-rp-ext:action?            enumeration
                    +--ro state
                       +--ro ip-prefix                    oc-inet:ip-prefix
                       +--ro masklength-range?            string
                       +--ro oc-rp-ext:sequence-number?   uint32
                       +--ro oc-rp-ext:action?            enumeration
```

# Definition/Abbreviation
### Table 1: Abbreviations
| **Term**                 | **Definition**                         |
|--------------------------|-------------------------------------|
| YANG                     | Yet Another Next Generation: modular language representing data structures in an XML tree format        |
| REST                     | REpresentative State Transfer |
| gNMI                     | gRPC Network Management Interface: used to retrieve or manipulate the state of a device via telemetry or configuration data         |
| ACL                      | Access Control List   |
| BGP                      | Border Gateway Protocol   |

# 1 Feature Overview
## 1.1 Requirements
### 1.1.1 Functional Requirements
1. Provide support for OpenConfig YANG models for Prefix List configuration.
2. Configure/Set, GET, and Delete prefix lists with IPv4 and IPv6 prefixes.
3. Support prefix matching with mask length ranges (e.g., "24..32" for /24 to /32).
4. Support exact prefix matching.
5. Support prefix list mode (IPv4, IPv6).
6. Support sequence numbers for prefix entries.
7. Support permit/deny actions for prefix entries (extension).
8. Support prefix list descriptions (extension).
9. Map OpenConfig prefix-sets model to SONiC PREFIX_SET and PREFIX tables.

### 1.1.2 Configuration and Management Requirements
The Prefix List configurations can be done via REST and gNMI. The implementation will return an error if a configuration is not allowed.

**Important Notes:**
- Prefix lists are used in routing policies (route maps) to match prefixes.
- Mask length range format: "exact" or "min..max" (e.g., "24..32").
- Sequence numbers determine the order of prefix evaluation.

## 1.2 Design Overview
### 1.2.1 Basic Approach
SONiC uses FRR (Free Range Routing) for routing policy implementation. This feature adds support for OpenConfig based YANG models using transformer based implementation in the Management Framework.

The implementation provides mapping between:
- OpenConfig prefix-sets config → SONiC PREFIX_SET table
- OpenConfig prefix entries → SONiC PREFIX table
- OpenConfig extensions → SONiC-specific prefix list features

### 1.2.2 Container
The code changes for this feature are part of *mgmt-framework* container which includes the REST server and *gnmi* container for gNMI support in *sonic-mgmt-common* repository.

# 2 Functionality
## 2.1 Target Deployment Use Cases
1. REST client through which the user can perform PATCH, DELETE, POST, PUT, and GET operations on Prefix List configuration paths.
2. gNMI client with support for capabilities, get, set, and subscribe operations based on the supported YANG models.
3. Prefix lists used as match conditions in routing policies (route maps).

# 3 Design
## 3.1 Overview
This HLD design is in line with the [Management Framework HLD](https://github.com/project-arlo/SONiC/blob/master/doc/mgmt/Management%20Framework.md)

The implementation uses transformer functions in `translib/transformer/xfmr_prefix_list.go` to map between OpenConfig and SONiC data models.

### 3.1.1 Mapping Details

#### Table 2: OpenConfig YANG to SONiC YANG Mapping

| OpenConfig YANG Node | SONiC YANG File | DB Name | Table:Field |
|---------------------|-----------------|---------|-------------|
| **prefix-sets/prefix-set** | | | |
| config/name | sonic-routing-policy.yang | CONFIG_DB | PREFIX_SET:`<name>` |
| config/mode | sonic-routing-policy.yang | CONFIG_DB | PREFIX_SET:mode |
| config/description | sonic-routing-policy.yang | CONFIG_DB | PREFIX_SET:description |
| **prefixes/prefix** | | | |
| config/ip-prefix | sonic-routing-policy.yang | CONFIG_DB | PREFIX:`<name>\|<seq>\|<ip-prefix>\|<masklength-range>` |
| config/masklength-range | sonic-routing-policy.yang | CONFIG_DB | PREFIX:`<name>\|<seq>\|<ip-prefix>\|<masklength-range>` |
| config/sequence-number | sonic-routing-policy.yang | CONFIG_DB | PREFIX:`<name>\|<seq>\|<ip-prefix>\|<masklength-range>` |
| config/action | sonic-routing-policy.yang | CONFIG_DB | PREFIX:action |

**Notes:**
- **Bold** entries indicate major feature categories/containers
- State nodes mirror their corresponding config nodes and are read-only
- PREFIX_SET table stores prefix-set metadata (name, mode, description)
- PREFIX table stores individual prefix entries with composite key
- Key format for PREFIX: `"<prefix-set-name>|<sequence>|<ip-prefix>|<masklength-range>"`
- Mode can be IPv4 or IPv6 (IPv4 and IPv6 are supported, MIXED not supported)
- Mask length range format: "exact" or "min..max" (e.g., "32..64")

## 3.2 DB Changes

### 3.2.1 CONFIG DB

The **sonic-routing-policy.yang** schema is used for prefix list configuration.

**CONFIG_DB Examples:**

**PREFIX_SET Table:**
```
PREFIX_SET|prefix-list-v4
  "mode": "IPv4"
  "description": "IPv4 prefix list for filtering"
```

**PREFIX Table:**
```
PREFIX|prefix-list-v4|10|10.0.0.0/8|8..16
  "action": "permit"

PREFIX|prefix-list-v4|20|192.168.0.0/16|exact
  "action": "deny"

PREFIX|prefix-list-v6|10|2001:db8::/32|32..48
  "action": "permit"
```

**FRR Configuration:**

Changes are made in FRR configuration to support prefix lists. The frrcfgd daemon subscribes to PREFIX_SET and PREFIX tables and configures corresponding FRR prefix-list commands.

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
- **openconfig-policy-types.yang**: Policy type definitions
- **openconfig-inet-types.yang**: IP address type definitions

### 3.3.2 OpenConfig Extensions

The following extensions are added to the standard OpenConfig routing-policy model in **openconfig-routing-policy-ext.yang**:

#### 1. Prefix Set Description
```yang
augment /oc-rpol:routing-policy/oc-rpol:defined-sets/oc-rpol:prefix-sets/
        oc-rpol:prefix-set/oc-rpol:config {
  leaf description {
    type string;
    description "Description for the prefix set";
  }
}
```

**Purpose**: Allows adding a human-readable description to prefix lists for documentation purposes.

**Usage**: Provide a descriptive string explaining the purpose of the prefix list.

#### 2. Prefix Sequence Number
```yang
augment /oc-rpol:routing-policy/oc-rpol:defined-sets/oc-rpol:prefix-sets/
        oc-rpol:prefix-set/oc-rpol:prefixes/oc-rpol:prefix/oc-rpol:config {
  leaf sequence-number {
    type uint32;
    description "Sequence number for prefix entry ordering";
  }
}
```

**Purpose**: Specifies the order in which prefix entries are evaluated.

**Usage**: Lower sequence numbers are evaluated first. Typically uses increments of 10 (10, 20, 30, etc.).

#### 3. Prefix Action
```yang
augment /oc-rpol:routing-policy/oc-rpol:defined-sets/oc-rpol:prefix-sets/
        oc-rpol:prefix-set/oc-rpol:prefixes/oc-rpol:prefix/oc-rpol:config {
  leaf action {
    type enumeration {
      enum permit {
        description "Permit matching prefixes";
      }
      enum deny {
        description "Deny matching prefixes";
      }
    }
    description "Action to take for matching prefix";
  }
}
```

**Purpose**: Defines whether matching prefixes should be permitted or denied.

**Usage**: Set to "permit" to allow matching prefixes, "deny" to block them.

### 3.3.3 REST API Support

#### 3.3.3.1 GET
Supported at all levels (container, list, and leaf).

**Example: GET all prefix sets**
```bash
curl -X GET -k "https://192.168.1.1/restconf/data/openconfig-routing-policy:routing-policy/defined-sets/prefix-sets" -H "accept: application/yang-data+json"
```

Response:
```json
{
  "openconfig-routing-policy:prefix-sets": {
    "prefix-set": [
      {
        "name": "prefix-list-v4",
        "config": {
          "name": "prefix-list-v4",
          "mode": "IPV4",
          "openconfig-routing-policy-ext:description": "IPv4 prefix list"
        }
      }
    ]
  }
}
```

**Example: GET specific prefix set**
```bash
curl -X GET -k "https://192.168.1.1/restconf/data/openconfig-routing-policy:routing-policy/defined-sets/prefix-sets/prefix-set=prefix-list-v4" -H "accept: application/yang-data+json"
```

#### 3.3.3.2 POST
Used to create new configuration. Supported at container and leaf levels.

**Example: POST a new prefix set with prefixes**
```bash
curl -X POST -k "https://192.168.1.1/restconf/data/openconfig-routing-policy:routing-policy/defined-sets/prefix-sets" \
  -H "accept: */*" \
  -H "Content-Type: application/yang-data+json" \
  -d '{
    "prefix-set": [
      {
        "name": "prefix-list-v4",
        "config": {
          "name": "prefix-list-v4",
          "mode": "IPV4",
          "openconfig-routing-policy-ext:description": "IPv4 prefix list for filtering"
        },
        "prefixes": {
          "prefix": [
            {
              "ip-prefix": "10.0.0.0/8",
              "masklength-range": "8..16",
              "config": {
                "ip-prefix": "10.0.0.0/8",
                "masklength-range": "8..16",
                "openconfig-routing-policy-ext:sequence-number": 10,
                "openconfig-routing-policy-ext:action": "permit"
              }
            }
          ]
        }
      }
    ]
  }'
```

**Example: POST a prefix with exact match**
```bash
curl -X POST -k "https://192.168.1.1/restconf/data/openconfig-routing-policy:routing-policy/defined-sets/prefix-sets/prefix-set=prefix-list-v4/prefixes" \
  -H "accept: */*" \
  -H "Content-Type: application/yang-data+json" \
  -d '{
    "prefix": [
      {
        "ip-prefix": "192.168.0.0/16",
        "masklength-range": "exact",
        "config": {
          "ip-prefix": "192.168.0.0/16",
          "masklength-range": "exact",
          "openconfig-routing-policy-ext:sequence-number": 20,
          "openconfig-routing-policy-ext:action": "deny"
        }
      }
    ]
  }'
```

**Example: POST IPv6 prefix list**
```bash
curl -X POST -k "https://192.168.1.1/restconf/data/openconfig-routing-policy:routing-policy/defined-sets/prefix-sets" \
  -H "accept: */*" \
  -H "Content-Type: application/yang-data+json" \
  -d '{
    "prefix-set": [
      {
        "name": "prefix-list-v6",
        "config": {
          "name": "prefix-list-v6",
          "mode": "IPV6",
          "openconfig-routing-policy-ext:description": "IPv6 prefix list"
        },
        "prefixes": {
          "prefix": [
            {
              "ip-prefix": "2001:db8::/32",
              "masklength-range": "32..48",
              "config": {
                "ip-prefix": "2001:db8::/32",
                "masklength-range": "32..48",
                "openconfig-routing-policy-ext:sequence-number": 10,
                "openconfig-routing-policy-ext:action": "permit"
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

**Example: PUT prefix set config**
```bash
curl -X PUT -k "https://192.168.1.1/restconf/data/openconfig-routing-policy:routing-policy/defined-sets/prefix-sets/prefix-set=prefix-list-v4/config" \
  -H "accept: */*" \
  -H "Content-Type: application/yang-data+json" \
  -d '{
    "config": {
      "name": "prefix-list-v4",
      "mode": "IPV4",
      "openconfig-routing-policy-ext:description": "Updated IPv4 prefix list"
    }
  }'
```

#### 3.3.3.4 PATCH
Used to modify specific fields without replacing entire configuration. Supported at all levels.

**Example: PATCH prefix description**
```bash
curl -X PATCH -k "https://192.168.1.1/restconf/data/openconfig-routing-policy:routing-policy/defined-sets/prefix-sets/prefix-set=prefix-list-v4/config/openconfig-routing-policy-ext:description" \
  -H "accept: */*" \
  -H "Content-Type: application/yang-data+json" \
  -d '{"openconfig-routing-policy-ext:description": "Modified description"}'
```

**Example: PATCH prefix action (extension)**
```bash
curl -X PATCH -k "https://192.168.1.1/restconf/data/openconfig-routing-policy:routing-policy/defined-sets/prefix-sets/prefix-set=prefix-list-v4/prefixes/prefix=10.0.0.0%2F8,8..16/config/openconfig-routing-policy-ext:action" \
  -H "accept: */*" \
  -H "Content-Type: application/yang-data+json" \
  -d '{"openconfig-routing-policy-ext:action": "deny"}'
```

#### 3.3.3.5 DELETE
Supported at all levels.

**Example: DELETE specific prefix**
```bash
curl -X DELETE -k "https://192.168.1.1/restconf/data/openconfig-routing-policy:routing-policy/defined-sets/prefix-sets/prefix-set=prefix-list-v4/prefixes/prefix=10.0.0.0%2F8,8..16" \
  -H "accept: */*"
```

**Example: DELETE entire prefix set**
```bash
curl -X DELETE -k "https://192.168.1.1/restconf/data/openconfig-routing-policy:routing-policy/defined-sets/prefix-sets/prefix-set=prefix-list-v4" \
  -H "accept: */*"
```

### 3.3.4 gNMI Support

#### 3.3.4.1 GET

**Example: GET prefix sets**
```bash
gnmi_get -insecure -logtostderr -username admin -password password \
  -target_addr localhost:8080 \
  -xpath /openconfig-routing-policy:routing-policy/defined-sets/prefix-sets
```

#### 3.3.4.2 SET

**Example: SET prefix set**
```bash
# Create prefix_set.json:
{
  "openconfig-routing-policy:prefix-set": [
    {
      "name": "prefix-list-v4",
      "config": {
        "name": "prefix-list-v4",
        "mode": "IPV4"
      }
    }
  ]
}

gnmi_set -insecure -logtostderr -username admin -password password \
  -target_addr localhost:8080 \
  -xpath_target OC-YANG \
  -update /openconfig-routing-policy:routing-policy/defined-sets/prefix-sets/prefix-set:@/tmp/prefix_set.json
```

#### 3.3.4.3 DELETE

**Example: DELETE prefix set**
```bash
gnmi_set -insecure -logtostderr -username admin -password password \
  -target_addr localhost:8080 \
  -xpath_target OC-YANG \
  -delete "/openconfig-routing-policy:routing-policy/defined-sets/prefix-sets/prefix-set[name=prefix-list-v4]"
```

#### 3.3.4.4 SUBSCRIBE

gNMI subscription is supported for monitoring configuration changes on prefix list parameters.

# 4 Error Handling

The implementation handles various error scenarios and returns appropriate error responses:

- **Prefix Set Not Found**: Attempting to configure on a non-existent prefix set will return an error.
- **Invalid Prefix Format**: Providing an invalid IP prefix format will return an error.
- **Invalid Mask Length Range**: Providing an invalid mask length range format will return an error.
- **Mode Mismatch**: Attempting to add IPv6 prefix to IPv4 prefix list (or vice versa) will return an error.
- **Duplicate Sequence Number**: Attempting to create a prefix entry with existing sequence number will return an error.
- **Invalid Action**: Providing an invalid action value will return an error.

# 5 Unit Test Cases

Comprehensive test cases are available in `translib/transformer/xfmr_prefix_list_test.go`.

## 5.1 Functional Test Cases

### 5.1.1 Prefix Set Configuration Tests

- POST IPv4 prefix set with description
- POST IPv6 prefix set
- GET all prefix sets
- GET specific prefix set
- PATCH prefix set description
- DELETE prefix set

### 5.1.2 Prefix Entry Tests

- POST prefix with mask length range (e.g., "8..16")
- POST prefix with exact match
- POST prefix with sequence number
- POST prefix with permit action
- POST prefix with deny action
- POST multiple prefixes in same set
- PATCH prefix action
- PATCH sequence number
- DELETE specific prefix
- GET all prefixes in set

### 5.1.3 IPv6 Prefix Tests

- POST IPv6 prefix set
- POST IPv6 prefix with mask length range
- POST IPv6 prefix with exact match
- GET IPv6 prefix set

### 5.1.4 Integration Tests

- Configure prefix set with multiple prefixes
- Configure both IPv4 and IPv6 prefix sets
- Modify existing prefix set entries
- GET entire prefix-sets container

## 5.2 Negative Test Cases

- POST prefix set with invalid name
- POST prefix with invalid IP format
- POST prefix with invalid mask length range format
- POST IPv6 prefix to IPv4 prefix set
- POST IPv4 prefix to IPv6 prefix set
- POST duplicate prefix entry
- POST prefix with duplicate sequence number
- DELETE non-existing prefix set
- DELETE non-existing prefix
- PATCH prefix with invalid action value

# 6 References

1. [OpenConfig Routing Policy YANG Model](https://github.com/openconfig/public/tree/master/release/models/policy)
2. [SONiC Management Framework HLD](https://github.com/project-arlo/SONiC/blob/master/doc/mgmt/Management%20Framework.md)
3. [SONiC Routing Policy Configuration](https://github.com/sonic-net/SONiC/wiki/Routing-Policy)
4. [RFC 4632 - Classless Inter-domain Routing (CIDR)](https://tools.ietf.org/html/rfc4632)
