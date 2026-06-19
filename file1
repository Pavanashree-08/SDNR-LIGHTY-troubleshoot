# Data Provider Module — Root Cause Analysis & Fix Plan

## Problem Summary

The data-provider APIs return **HTTP 204 No Content** for reads and **HTTP 500 Server Error** for writes. The darpan dashboard cannot retrieve any device connection information from lighty.

## Root Cause Analysis

### Issue 1: All RPC handlers return empty outputs (→ HTTP 204)

The current [LightyDataProvider.java](file:///home/pavanashree/Pavana/lighty-sdnr-lite/modules/iosmcn-data-provider/src/main/java/org/iosmcn/lighty/sdnr/dataprovider/LightyDataProvider.java) is explicitly a **"NoDb" stub**. Every RPC handler does this:

```java
// Line 138 — readStatus
ReadStatusOutput output = new ReadStatusOutputBuilder().build();  // ← empty!

// Line 196-197 — readNetworkElementConnectionList
ReadNetworkElementConnectionListOutput output =
    new ReadNetworkElementConnectionListOutputBuilder().build();  // ← empty!
```

The builders are called with **no data populated**, so the YANG serializer produces a response with no body → RESTCONF returns **204 No Content**.

> [!CAUTION]
> The code comments explicitly say: *"NoDb — no database to query"* and *"The actual NETCONF topology reading can be added in a follow-up."* This is the core problem — the follow-up was never done.

### Issue 2: Only 4 of 28 RPCs are registered (→ HTTP 500 for create/update/delete)

The YANG model `data-provider@2020-11-10.yang` defines **28 RPCs**, but only 4 are registered:

| Registered ✅ | Missing ❌ |
|---|---|
| `read-status` | `create-network-element-connection` |
| `read-faultcurrent-list` | `update-network-element-connection` |
| `read-inventory-list` | `delete-network-element-connection` |
| `read-network-element-connection-list` | `read-faultlog-list`, `read-cmlog-list`, `read-eventlog-list`, `read-connectionlog-list` |
| | `read-maintenance-list`, `create-maintenance`, `update-maintenance`, `delete-maintenance` |
| | `read-mediator-server-list`, `create-mediator-server`, `update-mediator-server`, `delete-mediator-server` |
| | `read-pmdata-*` (6 RPCs), `read-gui-cut-through-entry`, `read-tls-key-entry` |
| | `read-inventory-device-list` |

When darpan calls `create-network-element-connection`, no handler exists → MD-SAL throws an exception → Jetty returns **HTTP 500 Server Error**.

### Issue 3: No data source — even if handlers existed

The RPCs need a **data source** to return meaningful results. In ONAP SDN-R, this is Elasticsearch/OpenSearch. In this lighty-sdnr-lite version, there is no database — but **the NETCONF topology in the MD-SAL operational datastore** already contains all the connected device information. It just needs to be read and mapped to the data-provider YANG output format.

## Proposed Changes

### [MODIFY] [LightyDataProvider.java](file:///home/pavanashree/Pavana/lighty-sdnr-lite/modules/iosmcn-data-provider/src/main/java/org/iosmcn/lighty/sdnr/dataprovider/LightyDataProvider.java)

**Major changes:**

#### 1. `readNetworkElementConnectionList` — Read live data from NETCONF topology

Instead of returning empty, read from `network-topology:network-topology/topology=topology-netconf` in the operational datastore:

- Use `lightyServices.getBindingDataBroker()` to read the NETCONF topology
- Map each `Node` in the topology to a `data-provider:network-element-connection-entity`:
  - `node-id` → `node.getNodeId().getValue()`
  - `host` → from `netconf-node-topology:host`
  - `port` → from `netconf-node-topology:port`
  - `status` → from `netconf-node-topology:connection-status` (mapped to ConnectionLogStatus enum)
  - `username` → from `netconf-node-topology:username` (if available)
- Return with proper pagination output

#### 2. `readStatus` — Compute live status from NETCONF topology

Read the same NETCONF topology and compute aggregate counts:
- Count nodes by connection status (Connected, Connecting, UnableToConnect, etc.)
- Set fault counts to 0 (no fault store without a database)
- Return properly populated `status-entity` with `faults` and `network-element-connections` containers

#### 3. Register `create-network-element-connection` RPC

Implement by writing a NETCONF mount-point via the MD-SAL config datastore:
- Create a new node entry in `network-topology:network-topology/topology=topology-netconf`
- Populate it with the host, port, credentials from the RPC input
- This triggers lighty's NETCONF southbound to connect to the device

#### 4. Register `update-network-element-connection` RPC

Similar to create, but uses a merge operation on existing nodes.

#### 5. Register `delete-network-element-connection` RPC

Delete the node from the NETCONF topology config datastore, which triggers disconnect.

#### 6. Register remaining read RPCs as empty stubs

For RPCs that genuinely need a database (faultlog, cmlog, pmdata, etc.), register proper stubs that return empty lists with valid pagination instead of no-content, so they don't return 500 errors.

## Open Questions

> [!IMPORTANT]
> **Q1:** Do you want me to implement all 28 RPCs, or focus on the critical ones needed by darpan? Based on your logs, the dashboard calls `read-status`, `read-network-element-connection-list`, and `create-network-element-connection`. I recommend implementing those 3 fully + registering the rest as empty stubs.

> [!IMPORTANT]  
> **Q2:** For `create-network-element-connection` — should it only create the NETCONF mountpoint (so the device auto-connects), or do you also need it to store metadata like `device-type` / `device-function` somewhere? Without a database, extra metadata beyond what NETCONF topology tracks would be lost on restart.

> [!IMPORTANT]
> **Q3:** Should `read-status` fault counters always return 0, or do you want me to also scan the current alarms from connected devices via NETCONF?

## Verification Plan

### Automated Tests
```bash
# After rebuild and restart, test read-network-element-connection-list
curl -s -u admin:admin -X POST http://localhost:8888/rests/operations/data-provider:read-network-element-connection-list \
  -H "Content-Type: application/yang-data+json" \
  -H "Accept: application/yang-data+json" \
  -d '{"input":{"pagination":{"size":10000,"page":1}}}' | python3 -m json.tool

# Test read-status  
curl -s -u admin:admin -X POST http://localhost:8888/rests/operations/data-provider:read-status \
  -H "Content-Type: application/yang-data+json" \
  -H "Accept: application/yang-data+json" \
  -d '{"input":{}}' | python3 -m json.tool

# Test create-network-element-connection (should mount device)
curl -s -u admin:admin -X POST http://localhost:8888/rests/operations/data-provider:create-network-element-connection \
  -H "Content-Type: application/yang-data+json" \
  -H "Accept: application/yang-data+json" \
  -d '{"input":{"node-id":"test-device","host":"172.20.0.5","port":830,"username":"netconf","password":"netconf!"}}' | python3 -m json.tool
```

### Expected Results
- `read-network-element-connection-list` → JSON with `data` list containing connected devices and `pagination` block
- `read-status` → JSON with `data` list containing fault counts and connection state counts
- `create-network-element-connection` → HTTP 200 with the created entity echoed back, device starts connecting
