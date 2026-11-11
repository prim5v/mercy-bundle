# Mercy network data models

> Conceptual logical tables for discovery, direct neighbors, topology knowledge and routing. These are design artifacts, not a finalized persistence schema.

## Why five separate views?

A network node learns information at different levels of certainty:

1. A radio scan observes an advertisement.
2. A handshake may establish a direct peer relationship.
3. A peer may advertise nodes that are not directly reachable.
4. Route computation determines how this node could reach a destination.

Combining these concepts into one table makes stale observations, direct links and learned routes difficult to distinguish.

## 1. Discovery table

Short-lived observations emitted by local scans or received advertisements.

| Field | Example | Meaning |
|---|---|---|
| observation_id | D-1042 | Unique scan observation |
| observed_node_id | node-B | Peer identifier |
| discovered_at | 2026-11-11T09:20:00Z | Observation time |
| rssi_dbm | -62 | Received signal strength |
| advertisement_type | mercy-peer | Advertisement class |
| protocol_version | 1 | Advertised protocol |
| expires_at | 2026-11-11T09:20:30Z | Observation expiry |

A discovery record means “this peer was observed.” It does not mean a stable connection exists.

## 2. Neighbor table

Current direct relationships with peers for which a usable local link or recent handshake exists.

| Field | Example | Meaning |
|---|---|---|
| neighbor_id | N-21 | Local neighbor record |
| node_id | node-B | Direct peer |
| link_state | CONNECTED | Current link state |
| transport | BLE_CONTROL | Link used for control |
| rssi_dbm | -62 | Latest signal reading |
| last_seen_at | 2026-11-11T09:20:12Z | Last successful contact |
| capabilities | [RELAY, WIFI_P2P] | Negotiated capabilities |
| expires_at | 2026-11-11T09:21:12Z | Neighbor timeout |

## 3. Neighbor routing table

Routes whose **next hop is a direct neighbor**. This table is useful for forwarding decisions at the current node.

| Field | Example | Meaning |
|---|---|---|
| destination_node_id | node-D | Final destination |
| next_hop_node_id | node-B | Direct neighbor to forward to |
| hop_count | 2 | Total path hops |
| path_cost | 18.4 | Illustrative route cost |
| route_source | ADVERTISED | How route was learned |
| sequence | 108 | Route freshness/version |
| expires_at | 2026-11-11T09:21:00Z | Route expiry |

## 4. Known network nodes table

A topology view of nodes learned through direct neighbors or route advertisements. These nodes are not necessarily in direct radio range.

| Field | Example | Meaning |
|---|---|---|
| node_id | node-D | Known remote node |
| learned_via | node-B | Neighbor that advertised it |
| advertised_roles | [PROVIDER] | Roles advertised by node |
| reachable | true | Last known reachability |
| last_updated_at | 2026-11-11T09:20:10Z | Last topology update |
| topology_sequence | 108 | Freshness/version |
| expires_at | 2026-11-11T09:22:10Z | Known-node expiry |

## 5. Known network nodes routing table

Candidate end-to-end paths through the network. This is broader than the direct-neighbor forwarding table.

| Field | Example | Meaning |
|---|---|---|
| destination_node_id | node-D | Destination or gateway |
| next_hop_node_id | node-B | First hop from this node |
| path | [local, B, C, D] | Known path, if retained |
| hop_count | 3 | Path length |
| total_cost | 27.6 | Illustrative accumulated cost |
| gateway_available | true | Whether destination offers egress |
| sequence | 108 | Route freshness |
| expires_at | 2026-11-11T09:22:00Z | Route expiry |

## Example topology

The following is fictional sample data for explaining the distinction.

Suppose node-A is our local device. It directly discovers node-B and node-C. Node-B reports that it can reach node-D directly through its own next hop. Node-D advertises provider availability.

### Discovery observations at node-A

| Observation | Node | RSSI | Seen at |
|---|---|---:|---|
| D-01 | node-B | -62 dBm | 09:20:00 |
| D-02 | node-C | -71 dBm | 09:20:02 |
| D-03 | node-B | -60 dBm | 09:20:08 |

### Direct neighbors at node-A

| Neighbor | Node | Link state | Control transport |
|---|---|---|---|
| N-01 | node-B | CONNECTED | BLE_CONTROL |
| N-02 | node-C | CONNECTED | BLE_CONTROL |

### Neighbor routing view at node-A

| Destination | Next hop | Hops | Cost |
|---|---|---:|---:|
| node-B | node-B | 1 | 4.2 |
| node-C | node-C | 1 | 6.1 |
| node-D | node-B | 2 | 12.8 |

### Known network nodes at node-A

| Node | Learned via | Role | Direct neighbor? |
|---|---|---|---|
| node-B | local discovery | RELAY | Yes |
| node-C | local discovery | RELAY | Yes |
| node-D | node-B | PROVIDER | No |

### Known network routes at node-A

| Destination | Next hop | Path | Hops | Cost |
|---|---|---|---:|---:|
| node-B | node-B | A → B | 1 | 4.2 |
| node-C | node-C | A → C | 1 | 6.1 |
| node-D | node-B | A → B → D | 2 | 12.8 |

All values are illustrative. The route A → B → D is valid only if B has a usable next hop to D; a topology advertisement alone does not prove the data path is currently usable.

## Data lifecycle and consistency rules

- Discovery observations should have short TTLs and be overwritten or expired as fresh scans arrive.
- A neighbor should be removed or marked stale after missed handshakes or link failure.
- Learned remote nodes should expire when their advertisements become stale.
- Routes should be invalidated when their next hop disappears, the route sequence changes, the provider withdraws availability, or the transport fails.
- A node should reject route updates that create loops or are older than the currently accepted sequence for the same origin.
- Route cost should be calculated from explicit metrics; example costs above are not a prescribed formula.

## Open design questions

- Will each node store complete paths or only next-hop plus destination?
- Which route algorithm is appropriate for a mobile, intermittently connected network?
- How should sequence numbers and route withdrawals work?
- How are provider capabilities authenticated and refreshed?
- Which metrics are cheap and reliable enough to use on mobile devices?
- Should topology state be held entirely in memory, persisted selectively, or both?

These questions should be answered with small test networks and measured behavior before finalizing schemas.
