# OpenConfig support for BGP

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
This document provides general information about the OpenConfig configuration of BGP (Border Gateway Protocol) in SONiC.

# Scope
- This document describes the high level design of BGP configuration using OpenConfig models via REST & gNMI.
- This does not cover the SONiC KLISH CLI.
- This covers BGP global configuration, neighbors, peer-groups, AFI/SAFI configuration, graceful restart, route selection, and aggregates.
- This includes OpenConfig extensions for BGP features specific to SONiC/FRR implementation.
- Supported attributes in OpenConfig YANG tree (see below).

# Definition/Abbreviation
### Table 1: Abbreviations
| **Term**                 | **Definition**                         |
|--------------------------|-------------------------------------|
| YANG                     | Yet Another Next Generation: modular language representing data structures in an XML tree format        |
| REST                     | REpresentative State Transfer |
| gNMI                     | gRPC Network Management Interface: used to retrieve or manipulate the state of a device via telemetry or configuration data         |
| BGP                      | Border Gateway Protocol   |
| AS                       | Autonomous System   |
| AFI                      | Address Family Identifier   |
| SAFI                     | Subsequent Address Family Identifier   |
| EBGP                     | External BGP   |
| IBGP                     | Internal BGP   |
| BFD                      | Bidirectional Forwarding Detection   |
| VRF                      | Virtual Routing and Forwarding   |

# 1 Feature Overview
## 1.1 Requirements
### 1.1.1 Functional Requirements
1. Provide support for OpenConfig YANG models for BGP configuration.
2. Configure/Set, GET, and Delete BGP global parameters (AS, router-ID).
3. Support BGP neighbor configuration with peer-AS and description.
4. Support BGP peer-group configuration and templates.
5. Support AFI/SAFI configuration for IPv4/IPv6 unicast.
6. Support graceful restart configuration.
7. Support route selection options and use-multiple-paths.
8. Support BGP timers (hold-time, keepalive-interval).
9. Support BFD enablement for neighbors and peer-groups.
10. Support local aggregates (network statements).
11. Support dynamic neighbor configuration with prefix ranges.
12. Support table connections for route redistribution.
13. Support OpenConfig extensions for SONiC-specific BGP features.
14. Map OpenConfig BGP model to SONiC BGP CONFIG_DB tables.

### 1.1.2 Configuration and Management Requirements
The BGP configurations can be done via REST and gNMI. The implementation will return an error if a configuration is not allowed.

**Important Notes:**
- BGP configuration is under network-instance/protocols/protocol with identifier=BGP.
- State information includes session state, prefix counts, and operational status.
- Extensions are used for SONiC-specific features not in standard OpenConfig.

## 1.2 Design Overview
### 1.2.1 Basic Approach
SONiC uses FRR (Free Range Routing) for BGP protocol implementation. This feature adds support for OpenConfig based YANG models using transformer based implementation in the Management Framework.

The implementation provides mapping between:
- OpenConfig BGP global config → SONiC BGP_GLOBALS table
- OpenConfig BGP neighbors → SONiC BGP_NEIGHBOR table
- OpenConfig BGP peer-groups → SONiC BGP_PEER_GROUP table
- OpenConfig AFI/SAFI config → SONiC BGP address-family tables
- OpenConfig extensions → SONiC-specific BGP features

The FRR configuration is managed through the Unified FRR Management Interface framework.

### 1.2.2 Container
The code changes for this feature are part of *mgmt-framework* container which includes the REST server and *gnmi* container for gNMI support in *sonic-mgmt-common* repository.

# 2 Functionality
## 2.1 Target Deployment Use Cases
1. REST client through which the user can perform PATCH, DELETE, POST, PUT, and GET operations on BGP configuration paths.
2. gNMI client with support for capabilities, get, set, and subscribe operations based on the supported YANG models.
3. Telemetry streaming of BGP operational state (session state, prefix counts, etc.).

# 3 Design
## 3.1 Overview
This HLD design is in line with the [Management Framework HLD](https://github.com/project-arlo/SONiC/blob/master/doc/mgmt/Management%20Framework.md)

The implementation uses transformer functions in `translib/transformer/xfmr_bgp.go` to map between OpenConfig and SONiC data models.

### 3.1.1 Mapping Details

Due to the extensive nature of BGP configuration, the mapping table is organized by functional areas.

#### Table 2: OpenConfig YANG to SONiC YANG Mapping

| OpenConfig YANG Node | SONiC YANG File | DB Name | Table:Field |
|---------------------|-----------------|---------|-------------|
| **bgp/global** | | | |
| config/as | sonic-bgp-global.yang | CONFIG_DB | BGP_GLOBALS\|vrf_name:local_asn |
| config/router-id | sonic-bgp-global.yang | CONFIG_DB | BGP_GLOBALS\|vrf_name:router_id |
| **bgp/global/graceful-restart** | | | |
| config/enabled | sonic-bgp-global.yang | CONFIG_DB | BGP_GLOBALS\|vrf_name:graceful_restart_enable |
| config/restart-time | sonic-bgp-global.yang | CONFIG_DB | BGP_GLOBALS\|vrf_name:gr_restart_time |
| config/stale-routes-time | sonic-bgp-global.yang | CONFIG_DB | BGP_GLOBALS\|vrf_name:gr_stale_routes_time |
| config/max-dynamic-neighbor-prefixes (ext) | sonic-bgp-global.yang | CONFIG_DB | BGP_GLOBALS\|vrf_name:max_dynamic_neighbors |
| **bgp/global/use-multiple-paths/ebgp** | | | |
| config/allow-multiple-as | sonic-bgp-global.yang | CONFIG_DB | BGP_GLOBALS\|vrf_name:load_balance_mp_relax |
| **bgp/global/route-selection-options** | | | |
| config/external-compare-router-id | sonic-bgp-global.yang | CONFIG_DB | BGP_GLOBALS\|vrf_name:external_compare_router_id |
| **bgp/global/afi-safis/afi-safi** | | | |
| config/afi-safi-name | sonic-bgp-global.yang | CONFIG_DB | BGP_GLOBALS_AF\|vrf_name\|afi_safi:afi_safi_name |
| config/enabled | sonic-bgp-global.yang | CONFIG_DB | BGP_GLOBALS_AF\|vrf_name\|afi_safi:admin_status |
| state/total-paths | N/A | APPL_DB | BGP_GLOBALS_AF_TABLE:total_paths |
| state/total-prefixes | N/A | APPL_DB | BGP_GLOBALS_AF_TABLE:total_prefixes |
| **bgp/global/afi-safis/afi-safi/graceful-restart** | | | |
| config/enabled | sonic-bgp-global.yang | CONFIG_DB | BGP_GLOBALS_AF\|vrf_name\|afi_safi:graceful_restart_enable |
| **bgp/global/afi-safis/afi-safi/use-multiple-paths/ebgp** | | | |
| config/maximum-paths | sonic-bgp-global.yang | CONFIG_DB | BGP_GLOBALS_AF\|vrf_name\|afi_safi:max_ebgp_paths |
| **bgp/global/afi-safis/afi-safi/use-multiple-paths/ibgp** | | | |
| config/maximum-paths | sonic-bgp-global.yang | CONFIG_DB | BGP_GLOBALS_AF\|vrf_name\|afi_safi:max_ibgp_paths |
| **bgp/global/afi-safis/afi-safi/network (ext)** | | | |
| config/prefix | sonic-bgp-global.yang | CONFIG_DB | BGP_GLOBALS_AF_NETWORK\|vrf_name\|afi_safi\|prefix:NULL |
| **bgp/global/dynamic-neighbor-prefixes/dynamic-neighbor-prefix** | | | |
| config/prefix | sonic-bgp-global.yang | CONFIG_DB | BGP_GLOBALS_LISTEN_PREFIX\|vrf_name\|ip_prefix:NULL |
| config/peer-group | sonic-bgp-global.yang | CONFIG_DB | BGP_GLOBALS_LISTEN_PREFIX\|vrf_name\|ip_prefix:peer_group_name |
| **bgp/neighbors/neighbor** | | | |
| config/neighbor-address | sonic-bgp-neighbor.yang | CONFIG_DB | BGP_NEIGHBOR\|vrf_name\|neighbor_addr:NULL |
| config/peer-as | sonic-bgp-neighbor.yang | CONFIG_DB | BGP_NEIGHBOR\|vrf_name\|neighbor_addr:asn |
| config/peer-group | sonic-bgp-neighbor.yang | CONFIG_DB | BGP_NEIGHBOR\|vrf_name\|neighbor_addr:peer_group_name |
| config/description | sonic-bgp-neighbor.yang | CONFIG_DB | BGP_NEIGHBOR\|vrf_name\|neighbor_addr:name |
| config/enabled | sonic-bgp-neighbor.yang | CONFIG_DB | BGP_NEIGHBOR\|vrf_name\|neighbor_addr:admin_status |
| state/session-state | N/A | STATE_DB | NEIGH_STATE_TABLE\|vrf_name\|neighbor_addr:state (read-only) |
| state/last-established | N/A | STATE_DB | NEIGH_STATE_TABLE\|vrf_name\|neighbor_addr:uptime (read-only) |
| state/received-prefixes-count (ext) | N/A | STATE_DB | NEIGH_STATE_TABLE\|vrf_name\|neighbor_addr:rcvdPfx (read-only) |
| **bgp/neighbors/neighbor/timers** | | | |
| config/hold-time | sonic-bgp-neighbor.yang | CONFIG_DB | BGP_NEIGHBOR\|vrf_name\|neighbor_addr:holdtime |
| config/keepalive-interval | sonic-bgp-neighbor.yang | CONFIG_DB | BGP_NEIGHBOR\|vrf_name\|neighbor_addr:keepalive |
| **bgp/neighbors/neighbor/afi-safis/afi-safi** | | | |
| config/afi-safi-name | sonic-bgp-neighbor.yang | CONFIG_DB | BGP_NEIGHBOR_AF\|vrf_name\|neighbor_addr\|afi_safi:afi_safi_name |
| config/enabled | sonic-bgp-neighbor.yang | CONFIG_DB | BGP_NEIGHBOR_AF\|vrf_name\|neighbor_addr\|afi_safi:admin_status |
| **bgp/neighbors/neighbor/afi-safis/afi-safi/apply-policy** | | | |
| config/import-policy | sonic-bgp-neighbor.yang | CONFIG_DB | BGP_NEIGHBOR_AF\|vrf_name\|neighbor_addr\|afi_safi:route_map_in (array) |
| config/export-policy | sonic-bgp-neighbor.yang | CONFIG_DB | BGP_NEIGHBOR_AF\|vrf_name\|neighbor_addr\|afi_safi:route_map_out (array) |
| **bgp/neighbors/neighbor/enable-bfd** | | | |
| config/enabled | sonic-bgp-neighbor.yang | CONFIG_DB | BGP_NEIGHBOR\|vrf_name\|neighbor_addr:bfd |
| config/desired-minimum-tx-interval | sonic-bgp-neighbor.yang | CONFIG_DB | BGP_NEIGHBOR\|vrf_name\|neighbor_addr:desired_minimum_tx_interval |
| config/required-minimum-receive | sonic-bgp-neighbor.yang | CONFIG_DB | BGP_NEIGHBOR\|vrf_name\|neighbor_addr:required_minimum_receive |
| config/detection-multiplier | sonic-bgp-neighbor.yang | CONFIG_DB | BGP_NEIGHBOR\|vrf_name\|neighbor_addr:detection_multiplier |
| **bgp/peer-groups/peer-group** | | | |
| config/peer-group-name | sonic-bgp-peergroup.yang | CONFIG_DB | BGP_PEER_GROUP\|vrf_name\|peer_group:NULL |
| config/peer-as | sonic-bgp-peergroup.yang | CONFIG_DB | BGP_PEER_GROUP\|vrf_name\|peer_group:asn |
| config/description | sonic-bgp-peergroup.yang | CONFIG_DB | BGP_PEER_GROUP\|vrf_name\|peer_group:name |
| **bgp/peer-groups/peer-group/timers** | | | |
| config/hold-time | sonic-bgp-peergroup.yang | CONFIG_DB | BGP_PEER_GROUP\|vrf_name\|peer_group:holdtime |
| config/keepalive-interval | sonic-bgp-peergroup.yang | CONFIG_DB | BGP_PEER_GROUP\|vrf_name\|peer_group:keepalive |
| **bgp/peer-groups/peer-group/transport** | | | |
| config/local-address | sonic-bgp-peergroup.yang | CONFIG_DB | BGP_PEER_GROUP\|vrf_name\|peer_group:local_addr |
| **bgp/peer-groups/peer-group/ebgp-multihop** | | | |
| config/enabled | sonic-bgp-peergroup.yang | CONFIG_DB | BGP_PEER_GROUP\|vrf_name\|peer_group:ebgp_multihop |
| config/multihop-ttl | sonic-bgp-peergroup.yang | CONFIG_DB | BGP_PEER_GROUP\|vrf_name\|peer_group:ebgp_multihop_ttl |
| **bgp/peer-groups/peer-group/afi-safis/afi-safi** | | | |
| config/afi-safi-name | sonic-bgp-peergroup.yang | CONFIG_DB | BGP_PEER_GROUP_AF\|vrf_name\|peer_group\|afi_safi:afi_safi_name |
| config/enabled | sonic-bgp-peergroup.yang | CONFIG_DB | BGP_PEER_GROUP_AF\|vrf_name\|peer_group\|afi_safi:admin_status |
| **bgp/peer-groups/peer-group/afi-safis/afi-safi/apply-policy** | | | |
| config/import-policy | sonic-bgp-peergroup.yang | CONFIG_DB | BGP_PEER_GROUP_AF\|vrf_name\|peer_group\|afi_safi:route_map_in (array) |
| config/export-policy | sonic-bgp-peergroup.yang | CONFIG_DB | BGP_PEER_GROUP_AF\|vrf_name\|peer_group\|afi_safi:route_map_out (array) |
| **bgp/peer-groups/peer-group/enable-bfd** | | | |
| config/enabled | sonic-bgp-peergroup.yang | CONFIG_DB | BGP_PEER_GROUP\|vrf_name\|peer_group:bfd |
| config/desired-minimum-tx-interval | sonic-bgp-peergroup.yang | CONFIG_DB | BGP_PEER_GROUP\|vrf_name\|peer_group:desired_minimum_tx_interval |
| config/required-minimum-receive | sonic-bgp-peergroup.yang | CONFIG_DB | BGP_PEER_GROUP\|vrf_name\|peer_group:required_minimum_receive |
| config/detection-multiplier | sonic-bgp-peergroup.yang | CONFIG_DB | BGP_PEER_GROUP\|vrf_name\|peer_group:detection_multiplier |
| **protocols/protocol/local-aggregates/aggregate** | | | |
| config/prefix | sonic-bgp-global.yang | CONFIG_DB | BGP_GLOBALS_AF_AGGREGATE_ADDR\|vrf_name\|afi_safi\|prefix:NULL |
| config/summary-only | sonic-bgp-global.yang | CONFIG_DB | BGP_GLOBALS_AF_AGGREGATE_ADDR\|vrf_name\|afi_safi\|prefix:summary_only |
| **table-connections/table-connection** | | | |
| config/import-policy | sonic-protocol.yang | CONFIG_DB | ROUTE_REDISTRIBUTE\|vrf_name\|src_protocol\|dst_protocol\|addr_family:route_map |

**Notes:**
- **Bold** entries indicate major feature categories/containers
- State nodes mirror their corresponding config nodes and are read-only
- BGP configuration is VRF-aware; all tables use `vrf_name` as part of the key
- **Key format examples:**
  - BGP_GLOBALS: `"default"` or `"Vrf1"`
  - BGP_NEIGHBOR: `"default|192.0.10.1"` or `"Vrf1|192.0.10.1"`
  - BGP_GLOBALS_AF: `"default|ipv4_unicast"` or `"Vrf1|ipv6_unicast"`
  - BGP_GLOBALS_AF_NETWORK: `"default|ipv4_unicast|10.1.1.0/24"`
  - ROUTE_REDISTRIBUTE: `"default|connected|bgp|ipv4"` (lowercase protocol and addr_family)
- AFI/SAFI identifiers: `ipv4_unicast`, `ipv6_unicast`, `l2vpn_evpn` (lowercase with underscores)
- Protocol identifiers for table-connections: `connected`, `static`, `bgp`, `ospf` (lowercase)
- Peer-group configuration serves as a template for neighbors
- BFD configuration fields are on BGP_NEIGHBOR and BGP_PEER_GROUP tables
- Policy fields (route_map_in/route_map_out) are arrays supporting multiple policies
- **Table-connections (route redistribution):** Uses ROUTE_REDISTRIBUTE table (not PROTOCOL_ROUTE_MAP)
- Dynamic neighbor prefixes use BGP_GLOBALS_LISTEN_PREFIX table
- Local aggregates use BGP_GLOBALS_AF_AGGREGATE_ADDR table
- **BGP network prefixes (extension):** Use BGP_GLOBALS_AF_NETWORK table for advertised networks

## 3.2 DB Changes

### 3.2.1 CONFIG DB

The **sonic-bgp-global.yang**, **sonic-bgp-neighbor.yang**, **sonic-bgp-peergroup.yang**, and **sonic-bgp-peerrange.yang** schemas are used for BGP configuration.

### 3.2.2 APPL_DB
BGP operational state for global AFI-SAFI statistics is available in APPL_DB:

**BGP_GLOBALS_AF_TABLE:**
```json
{
  "BGP_GLOBALS_AF_TABLE:default:ipv4_unicast": {
    "total_paths": "1000",
    "total_prefixes": "950"
  }
}
```

**Note:** BGP neighbor state information (session-state, last-established, received-prefixes-count) is stored in STATE_DB NEIGH_STATE_TABLE, not APPL_DB.

### 3.2.3 STATE DB
BGP neighbor operational state is available in STATE_DB:

**NEIGH_STATE_TABLE:**
```json
{
  "NEIGH_STATE_TABLE:default|10.1.0.2": {
    "state": "Established",
    "peerType": "e-BGP",
    "vrf": "default",
    "rcvdPfx": "150",
    "uptime": "360000"
  }
}
```

**Key Format:** `vrf_name|neighbor_ip`

**Fields:**
- `state`: BGP session state (Idle, Connect, Active, OpenSent, OpenConfirm, Established)
- `peerType`: Peer type (e-BGP, i-BGP)
- `vrf`: VRF name
- `rcvdPfx`: Total number of prefixes received from neighbor (aggregate across all AFI-SAFIs)
- `uptime`: Time since session was established (in centiseconds, timeticks64 format)

### 3.2.4 ASIC DB
There are no changes to ASIC DB schema definition.

### 3.2.5 COUNTER DB
There are no changes to COUNTER DB schema definition.

## 3.3 User Interface
### 3.3.1 Data Models
The implementation uses standard OpenConfig YANG models:
- **openconfig-network-instance.yang**: Main network instance model
- **openconfig-bgp.yang**: BGP protocol definitions
- **openconfig-bgp-types.yang**: BGP type definitions
- **openconfig-routing-policy.yang**: Policy references

### 3.3.2 OpenConfig Extensions

The following extensions are added to the standard OpenConfig BGP model in **openconfig-network-instance-ext.yang**:

#### 1. Enable Default IPv4 Unicast AFI
```yang
augment /oc-ni:network-instances/oc-ni:network-instance/oc-ni:protocols/
        oc-ni:protocol/oc-ni:bgp/oc-ni:global/oc-ni:config {
  leaf enable-default-ipv4-unicast-afi {
    type boolean;
    description "Enable default IPv4 unicast address family for all neighbors";
  }
}
```

#### 2. Max Dynamic Neighbor Prefixes
```yang
augment /oc-ni:network-instances/oc-ni:network-instance/oc-ni:protocols/
        oc-ni:protocol/oc-ni:bgp/oc-ni:global/oc-ni:config {
  leaf max-dynamic-neighbor-prefixes {
    type uint16;
    description "Maximum number of dynamic neighbor prefixes";
  }
}
```

#### 3. Graceful Restart Extensions
```yang
augment /oc-ni:network-instances/oc-ni:network-instance/oc-ni:protocols/
        oc-ni:protocol/oc-ni:bgp/oc-ni:global/oc-ni:graceful-restart/oc-ni:config {
  leaf preserve-fw-state {
    type boolean;
    description "Preserve forwarding state during restart";
  }
  leaf disable-end-of-rib-marker {
    type boolean;
    description "Disable end-of-RIB marker";
  }
}
```

#### 4. Route Selection Network Import Check
```yang
augment /oc-ni:network-instances/oc-ni:network-instance/oc-ni:protocols/
        oc-ni:protocol/oc-ni:bgp/oc-ni:global/oc-ni:route-selection-options/oc-ni:config {
  leaf network-import-check {
    type boolean;
    description "Check if BGP network route is present in routing table";
  }
}
```

#### 5. AFI/SAFI Import Policy
```yang
augment /oc-ni:network-instances/oc-ni:network-instance/oc-ni:protocols/
        oc-ni:protocol/oc-ni:bgp/oc-ni:global/oc-ni:afi-safis/oc-ni:afi-safi/oc-ni:config {
  leaf-list import-policy {
    type leafref {
      path /oc-rpol:routing-policy/oc-rpol:policy-definitions/oc-rpol:policy-definition/oc-rpol:name;
    }
    description "Import policy for this address family";
  }
}
```

#### 6. BGP Network Statement
```yang
augment /oc-ni:network-instances/oc-ni:network-instance/oc-ni:protocols/
        oc-ni:protocol/oc-ni:bgp/oc-ni:global/oc-ni:afi-safis/oc-ni:afi-safi {
  container networks {
    list network {
      key "prefix";
      leaf prefix {
        type leafref {
          path "../config/prefix";
        }
      }
      container config {
        leaf prefix {
          type inet:ip-prefix;
          description "Network prefix to advertise";
        }
      }
      container state {
        config false;
        leaf prefix {
          type inet:ip-prefix;
        }
      }
    }
  }
}
```

#### 7. Aggregate Summary Only
```yang
augment /oc-ni:network-instances/oc-ni:network-instance/oc-ni:protocols/
        oc-ni:protocol/oc-ni:local-aggregates/oc-ni:aggregate/oc-ni:config {
  leaf summary-only {
    type boolean;
    description "Advertise only the aggregate, suppress more specific routes";
  }
}
```

#### 8. EBGP Requires Policy
```yang
augment /oc-ni:network-instances/oc-ni:network-instance/oc-ni:protocols/
        oc-ni:protocol/oc-ni:bgp/oc-ni:global/oc-ni:apply-policy/oc-ni:config {
  leaf ebgp-requires-policy {
    type boolean;
    description "Require import/export policy for EBGP sessions";
  }
}
```

#### 9. BGP Logging Options
```yang
augment /oc-ni:network-instances/oc-ni:network-instance/oc-ni:protocols/
        oc-ni:protocol/oc-ni:bgp/oc-ni:global {
  container logging-options {
    container config {
      leaf log-neighbor-state-changes {
        type boolean;
        description "Log BGP neighbor state changes";
      }
    }
    container state {
      config false;
      leaf log-neighbor-state-changes {
        type boolean;
      }
    }
  }
}
```

#### 10. BGP Timers (Global)
```yang
augment /oc-ni:network-instances/oc-ni:network-instance/oc-ni:protocols/
        oc-ni:protocol/oc-ni:bgp/oc-ni:global {
  container timers {
    container config {
      leaf hold-time {
        type uint16;
        description "BGP hold time";
      }
      leaf keepalive-interval {
        type uint16;
        description "BGP keepalive interval";
      }
    }
    container state {
      config false;
      leaf hold-time {
        type uint16;
      }
      leaf keepalive-interval {
        type uint16;
      }
    }
  }
}
```

#### 11. Neighbor Received Prefixes Count
```yang
augment /oc-ni:network-instances/oc-ni:network-instance/oc-ni:protocols/
        oc-ni:protocol/oc-ni:bgp/oc-ni:neighbors/oc-ni:neighbor/oc-ni:state {
  leaf received-prefixes-count {
    type uint32;
    description "Number of prefixes received from neighbor";
  }
}
```

#### 12. Soft Reconfiguration Inbound
```yang
augment /oc-ni:network-instances/oc-ni:network-instance/oc-ni:protocols/
        oc-ni:protocol/oc-ni:bgp/oc-ni:neighbors/oc-ni:neighbor/oc-ni:afi-safis/
        oc-ni:afi-safi/oc-ni:config {
  leaf soft-reconfiguration-inbound {
    type boolean;
    description "Enable soft reconfiguration for inbound updates";
  }
}
```

#### 13. Next-hop Self
```yang
augment /oc-ni:network-instances/oc-ni:network-instance/oc-ni:protocols/
        oc-ni:protocol/oc-ni:bgp/oc-ni:neighbors/oc-ni:neighbor/oc-ni:afi-safis/
        oc-ni:afi-safi/oc-ni:apply-policy/oc-ni:config {
  leaf next-hop-self {
    type boolean;
    description "Set next-hop to self for advertised routes";
  }
}
```

#### 14. Peer-group Capability Extended Nexthop
```yang
augment /oc-ni:network-instances/oc-ni:network-instance/oc-ni:protocols/
        oc-ni:protocol/oc-ni:bgp/oc-ni:peer-groups/oc-ni:peer-group/oc-ni:config {
  leaf capability-ext-nexthop {
    type boolean;
    description "Enable extended next-hop capability";
  }
}
```

### 3.3.3 REST API Support

#### 3.3.3.1 GET
Supported at all levels (container, list, and leaf).

**Example: GET BGP global config**
```bash
curl -X GET -k "https://192.168.1.1/restconf/data/openconfig-network-instance:network-instances/network-instance=default/protocols/protocol=BGP,bgp/bgp/global/config" -H "accept: application/yang-data+json"
```

Response:
```json
{
  "openconfig-bgp:config": {
    "as": 65001,
    "router-id": "1.1.1.1"
  }
}
```

**Example: GET BGP neighbor state**
```bash
curl -X GET -k "https://192.168.1.1/restconf/data/openconfig-network-instance:network-instances/network-instance=default/protocols/protocol=BGP,bgp/bgp/neighbors/neighbor=10.0.0.1/state" -H "accept: application/yang-data+json"
```

#### 3.3.3.2 POST
Used to create new configuration.

**Example: POST BGP global configuration**
```bash
curl -X POST -k "https://192.168.1.1/restconf/data/openconfig-network-instance:network-instances/network-instance=default/protocols" \
  -H "accept: */*" \
  -H "Content-Type: application/yang-data+json" \
  -d '{
    "protocol": [
      {
        "identifier": "BGP",
        "name": "bgp",
        "config": {
          "identifier": "BGP",
          "name": "bgp"
        },
        "bgp": {
          "global": {
            "config": {
              "as": 65001,
              "router-id": "1.1.1.1"
            }
          }
        }
      }
    ]
  }'
```

**Example: POST BGP neighbor**
```bash
curl -X POST -k "https://192.168.1.1/restconf/data/openconfig-network-instance:network-instances/network-instance=default/protocols/protocol=BGP,bgp/bgp/neighbors" \
  -H "accept: */*" \
  -H "Content-Type: application/yang-data+json" \
  -d '{
    "neighbor": [
      {
        "neighbor-address": "10.0.0.1",
        "config": {
          "neighbor-address": "10.0.0.1",
          "peer-as": 65002,
          "description": "Peer Router"
        },
        "timers": {
          "config": {
            "hold-time": 180,
            "keepalive-interval": 60
          }
        }
      }
    ]
  }'
```

**Example: POST BGP peer-group with extension**
```bash
curl -X POST -k "https://192.168.1.1/restconf/data/openconfig-network-instance:network-instances/network-instance=default/protocols/protocol=BGP,bgp/bgp/peer-groups" \
  -H "accept: */*" \
  -H "Content-Type: application/yang-data+json" \
  -d '{
    "peer-group": [
      {
        "peer-group-name": "spine",
        "config": {
          "peer-group-name": "spine",
          "peer-type": "EXTERNAL",
          "openconfig-network-instance-ext:capability-ext-nexthop": true
        }
      }
    ]
  }'
```

#### 3.3.3.3 PUT
Used to replace entire configuration.

**Example: PUT BGP graceful restart config**
```bash
curl -X PUT -k "https://192.168.1.1/restconf/data/openconfig-network-instance:network-instances/network-instance=default/protocols/protocol=BGP,bgp/bgp/global/graceful-restart/config" \
  -H "accept: */*" \
  -H "Content-Type: application/yang-data+json" \
  -d '{
    "config": {
      "enabled": true,
      "restart-time": 120,
      "stale-routes-time": 360
    }
  }'
```

#### 3.3.3.4 PATCH
Used to modify specific fields.

**Example: PATCH BGP AS number**
```bash
curl -X PATCH -k "https://192.168.1.1/restconf/data/openconfig-network-instance:network-instances/network-instance=default/protocols/protocol=BGP,bgp/bgp/global/config/as" \
  -H "accept: */*" \
  -H "Content-Type: application/yang-data+json" \
  -d '{"as": 65100}'
```

**Example: PATCH neighbor with BFD**
```bash
curl -X PATCH -k "https://192.168.1.1/restconf/data/openconfig-network-instance:network-instances/network-instance=default/protocols/protocol=BGP,bgp/bgp/neighbors/neighbor=10.0.0.1/enable-bfd/config" \
  -H "accept: */*" \
  -H "Content-Type: application/yang-data+json" \
  -d '{
    "config": {
      "enabled": true,
      "desired-minimum-tx-interval": 300,
      "required-minimum-receive": 300,
      "detection-multiplier": 3
    }
  }'
```

#### 3.3.3.5 DELETE
Supported at all levels.

**Example: DELETE BGP neighbor**
```bash
curl -X DELETE -k "https://192.168.1.1/restconf/data/openconfig-network-instance:network-instances/network-instance=default/protocols/protocol=BGP,bgp/bgp/neighbors/neighbor=10.0.0.1" \
  -H "accept: */*"
```

### 3.3.4 gNMI Support

#### 3.3.4.1 GET

**Example: GET BGP global config**
```bash
gnmi_get -insecure -logtostderr -username admin -password password \
  -target_addr localhost:8080 \
  -xpath /openconfig-network-instance:network-instances/network-instance[name=default]/protocols/protocol[identifier=BGP][name=bgp]/bgp/global/config
```

#### 3.3.4.2 SET

**Example: SET BGP neighbor**
```bash
gnmi_set -insecure -logtostderr -username admin -password password \
  -target_addr localhost:8080 \
  -xpath_target OC-YANG \
  -update "/openconfig-network-instance:network-instances/network-instance[name=default]/protocols/protocol[identifier=BGP][name=bgp]/bgp/neighbors/neighbor[neighbor-address=10.0.0.1]/config/peer-as:65002"
```

#### 3.3.4.3 DELETE

**Example: DELETE BGP peer-group**
```bash
gnmi_set -insecure -logtostderr -username admin -password password \
  -target_addr localhost:8080 \
  -xpath_target OC-YANG \
  -delete "/openconfig-network-instance:network-instances/network-instance[name=default]/protocols/protocol[identifier=BGP][name=bgp]/bgp/peer-groups/peer-group[peer-group-name=spine]"
```

#### 3.3.4.4 SUBSCRIBE

gNMI subscription is supported for monitoring BGP operational state.

**Example: Subscribe to BGP neighbor state**
```bash
gnmic -a localhost:8080 -u admin -p password --insecure --target OC-YANG \
  sub --path "openconfig-network-instance:network-instances/network-instance[name=default]/protocols/protocol[identifier=BGP][name=bgp]/bgp/neighbors/neighbor[neighbor-address=10.0.0.1]/state"
```

# 4 Error Handling

The implementation handles various error scenarios and returns appropriate error responses:

- **Network Instance Not Found**: Attempting to configure BGP in a non-existent network instance will return an error.
- **Invalid AS Number**: Providing an invalid AS number will return an error.
- **Invalid Router ID**: Providing an invalid router-id format will return an error.
- **Invalid Neighbor Address**: Providing an invalid neighbor IP address will return an error.
- **Peer-group Not Found**: Attempting to reference a non-existent peer-group will return an error.
- **Invalid AFI/SAFI**: Providing an unsupported AFI/SAFI identifier will return an error.
- **Invalid Policy Reference**: Referencing a non-existent routing policy will return an error.
- **Duplicate Configuration**: Attempting to create duplicate neighbor or peer-group will return an error.

## 4.1 Unsupported Features

The following OpenConfig BGP features are not currently supported in SONiC:

- **Table Connections (Route Redistribution)**: Supported via ROUTE_REDISTRIBUTE table for import-policy configuration.
- **Send-Community Configuration**: The `send-community-type` field in neighbor AFI/SAFI configuration is not supported.
- **Add-Paths Configuration**: The `add-paths` container under peer-group AFI/SAFI is not fully supported.
- **Peer-group Peer-Type**: The `peer-type` field in peer-group configuration is not supported.
- **Peer-group Connect-Retry**: The `connect-retry` timer in peer-group configuration is not supported.
- **Peer-group Graceful Restart**: Graceful restart configuration at the peer-group level is not supported; it is only available at the global level.

Attempts to configure these unsupported features will return appropriate error messages indicating that the feature is not supported.

# 5 Unit Test Cases

Comprehensive test cases are available in the transformer test suite.

## 5.1 Functional Test Cases

### 5.1.1 BGP Global Configuration Tests

- POST BGP global config with AS and router-ID
- GET BGP global config
- PATCH BGP AS number
- PATCH router-ID
- DELETE BGP global config

### 5.1.2 Graceful Restart Tests

- POST graceful restart config
- PATCH restart-time
- PATCH stale-routes-time
- PATCH preserve-fw-state (extension)
- GET graceful restart state

### 5.1.3 Route Selection Tests

- POST route selection options
- PATCH external-compare-router-id
- PATCH network-import-check (extension)
- GET route selection state

### 5.1.4 AFI/SAFI Configuration Tests

- POST AFI/SAFI (IPv4 unicast)
- POST AFI/SAFI (IPv6 unicast)
- PATCH AFI/SAFI enabled
- POST maximum-paths (EBGP/IBGP)
- POST import-policy on AFI/SAFI (extension)
- GET AFI/SAFI state with prefix counts

### 5.1.5 BGP Neighbor Tests

- POST neighbor with peer-AS
- POST neighbor with peer-group reference
- PATCH neighbor description
- POST neighbor timers
- GET neighbor session state
- DELETE neighbor

### 5.1.6 Neighbor AFI/SAFI Tests

- POST neighbor AFI/SAFI config
- PATCH send-community-type
- POST import/export policies
- PATCH next-hop-self (extension)
- PATCH soft-reconfiguration-inbound (extension)

### 5.1.7 BFD Configuration Tests

- POST BFD on neighbor
- PATCH BFD timers
- POST BFD on peer-group
- GET BFD state

### 5.1.8 Peer-group Tests

- POST peer-group
- PATCH peer-type
- POST peer-group timers
- POST peer-group transport config
- POST peer-group graceful restart
- POST peer-group ebgp-multihop
- PATCH capability-ext-nexthop (extension)
- DELETE peer-group

### 5.1.9 Peer-group AFI/SAFI Tests

- POST peer-group AFI/SAFI
- POST add-paths configuration
- POST import/export policies
- GET peer-group AFI/SAFI state

### 5.1.10 Aggregate Configuration Tests

- POST local aggregate
- PATCH summary-only (extension)
- GET aggregate state
- DELETE aggregate

### 5.1.11 Network Statement Tests (Extension)

- POST network under AFI/SAFI
- GET network list
- DELETE network

### 5.1.12 Dynamic Neighbor Tests

- POST dynamic neighbor prefix
- POST peer-group reference
- GET dynamic neighbor state
- DELETE dynamic neighbor prefix

### 5.1.13 Integration Tests

- Configure BGP with neighbors and peer-groups
- Configure multiple AFI/SAFIs with policies
- Configure BGP in multiple VRFs
- Configure dynamic neighbors with peer-group template
- Monitor neighbor state changes via subscription

## 5.2 Negative Test Cases

- POST BGP with invalid AS number
- POST BGP with invalid router-ID format
- POST neighbor with invalid IP address
- POST neighbor referencing non-existent peer-group
- POST AFI/SAFI with invalid identifier
- POST policy reference to non-existent policy
- POST duplicate neighbor
- POST duplicate peer-group
- DELETE non-existing neighbor
- PATCH non-existing AFI/SAFI
- POST neighbor in non-existent VRF
- POST BFD with invalid timer values

# 6 References

1. [OpenConfig Network Instance YANG Model](https://github.com/openconfig/public/tree/master/release/models/network-instance)
2. [OpenConfig BGP YANG Model](https://github.com/openconfig/public/tree/master/release/models/bgp)
3. [SONiC Management Framework HLD](https://github.com/project-arlo/SONiC/blob/master/doc/mgmt/Management%20Framework.md)
4. [SONiC BGP Configuration](https://github.com/sonic-net/SONiC/wiki/BGP-Configuration)
5. [RFC 4271 - Border Gateway Protocol 4 (BGP-4)](https://tools.ietf.org/html/rfc4271)
6. [RFC 4724 - Graceful Restart Mechanism for BGP](https://tools.ietf.org/html/rfc4724)
