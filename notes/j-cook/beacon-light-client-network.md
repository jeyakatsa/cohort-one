# Beacon Chain Light Clients

<b>WIP: contributions/corrections welcome!</b>

## What is the purpose of a Beacon Chain light client?

Currently, there are only two options for accessing Ethereum: through a centralised service or by running a full Ethereum node. The former is vulnerable to failure, manipulation or censorship of the centralised provider, while the latter requires significant technical know-how and non-negligible computing resources. A light-client sits between these two end-members. It is fully decentralised and the user verifies their own data and syncs the blockchain first-hand, eliminating the need to trust a third party and expends minimal computational resource perhaps to the extent that the application could run invisibly on a normal laptop or on a mobile phone. However, the trade-off is that in order to stay light, the client has to request access to state data from the network rather than querying its own storage. The aim of this document is to explore two sources of this state data to direct the development of new light clients.

## Networking models for light clients

Beacon chain light clients aim to sync the head of the Beacon chain and request specific state data with minimal requirements on memory, cpu and bandwidth. A full beacon client has access to the full state of the chain at all times, whereas light clients only store sufficient data to verify the latest block(s) and have to make specific requests for more detailed data. This is what makes a light client light - it can operate with minimal resources because it outsources data storage to some other entity and onyl handles it when needed.

This means a light client needs access to a verifiable source of a) new block headers, and b) specific merkle branches that give access to particular data (e.g. a user may wish to query their own account balance by retrieving the merkle branch that has their account information as the leaf). Syncing the chain is achieved by tracking only the previous and current sync committees at each block.

One possibility is to make requests to Beacon nodes that store the whole state merkle tree, not just the state root. An inbound request from a light client would define the required data and the full node would return the merkle branch containing it. Note that pre-merge all the only data available for a Beacon client to query is data that lives on the Beacon chain, for example validator deposit balances and consensus details. Post-merge data from the execution layer should also be served on request to light clients.

For this node-as-server model, the light client must be able to connect to multiple Beacon nodes for liveness redundancy, and those Beacon nodes must be actively serving light clients. This model has failed in the past. In principle Geth supports the light client network LES, but so few clients choose to support it that the network is effectively non-functional. There are two nuances of the Beacon client architecture that suggest the light client network might be better served on the consensus layer than historically on the execution layer. One is the Altair network upgrade that makes it simple and computationally cheap to serve light clients with the basic information for syncing - sync committees are randomly selected to summarise the chain for light clients to sync. The second is the intention to make serving light clients opt-out rather than opt-in, meaning the default setting for a Beacon node would be to serve light clients. These changes may be sufficient to sustain a light client network served by Beacon nodes. 

However, it would benefit the network if the light clients obtained their data from gossip spread across the p2p network rather than relying on specific nodes to serve requests. The sync-committee would gossip the relevant data by default and light clients listen for it, similar to how transaction gossip is spread on the execution layer. This requires a new networking layer to be established for light clients and for Beacon nodes to inject data into it periodically. Developers taking this approach would likely bootstrap a light client p2p network by replicating elements of the main consensus layer gossip protocol. A req/resp protocol could then be implemented to serve specific data requests. This is clearly feasible, as it would be a reduced version of the p2p layer already supporting the consensus layer. Developers across the Ethereum ecosystem are likely to be familiar with the p2p network that light clients would participate in, perhaps widening the pool of potential light client developers and participants. As has been proven by past experience with les, sufficent community buy-in is critical for the survival of the light client network.

The request/response model is rather extractive on the part of the light client. Unless the computational load on the serving Beacon node is negligibly small there should be some incentivization (perhaps in the form of a micropayment or a block reward boost) for nodes that serve light clients. This then requires adjustments to the Beacon client protocols and suddenly serving the light client network does not seem so simple. It seems likely that serving block headers that include state roots and committee pubkeys could be sufficiently cheap that Beacon nodes could serve this altruistically, without requiring explicit incentives; however, specific data requests that require traversing the full Beacon state might require some reward to participating nodes. 

On the other hand, a new p2p layer that requires Beacon nodes to inject data that then spreads over a light-client gossip network sounds a lot like the Portal Network. The Portal Network uses bridge nodes to seed the network with data, then gossips it beween connected peers. The difference is that data storage on the Portal Network is achieved by repurposing the node DHTs (distributed hash tables) previously used for node discovery. The DHTs would store a subset of the data available on a full Beacon node so that any specific data request can be served by searching the network for the DHT containing it, rather than explicitly making requests to Beacon nodes. This network is specifically designed to connect resource-constrained devices on a self-contained p2p network providing snapshots of the main Ethereum network but not actively participating in it. The portal network promises very efficient access to data for light clients.

So far, the portal network has been developed with the execution layer in mind, and its application to the consensus layer has not been spec'd. The advantage of the portal network approach is a clear separation of concerns between those clients participating fully in the Ethereum network and those only requiring light access. Once the portal network surpasses some critical number of participating nodes it should be self sustaining and requiring minimal interaction with the main Ethereum network. The portal approach therefore does not completely eliminate the need for benevolent Beacon nodes, because it requires data injection from bridge nodes, but it does minimize the reliance on them by abstracting specific data requests from light clients away from Beacon nodes. 


## How did lodestar do it?

Lodestar's light client uses a Lodestar Beacon node as a server, which the light client discovers via a fixed url (no p2p discovery). Beacon block headers and updates are precomputed and served by http, proofs are generated on demand and served by http endpoints. All that can currently be served is data in the Beacon chain - validator deposit balances, committee IDs etc. It also lags the true head of the chain because it syncs to finalised blocks and does not implement a for choice rule.

The tweaks made to the Lodestar Beacon Node to allow it to serve light clients via http are explained in detail [here](https://hackmd.io/hsCz1G3BTyiwwJtjT4pe2Q?view#Producing-LightClientUpdates). Briefly, the node exposes several API endpoints that trigger updates to the light client's knowledge of the current state. The light client bootstraps into the network via a trusted checkpoint. This takes the form of a GET request to the Beacon node for an initial proof object that contains the `current_sync_committee`, `next_sync_committee`, `genesis_time` and `genesis_validators_root` for the most recently finalised checkpoint. This is verified against the `beaconBlockHeader.state_root` and the light client has a verified snapshot that it can sync from. Syncing is achieved via GET requests to the `best_updates` endpoint, with 'best' meaning most-voted and recent. This syncs the chain to the start of the most recent sync committee period, and can then be further synced to the latest finalized state via GET request to `update_finalized`. The processing behind these endpoints is consistent with [this spec](https://github.com/ethereum/consensus-specs/blob/dev/specs/altair/sync-protocol.md).

Likely next steps for lodestar 
(I am totally speculating based on what I think is sensible - I have no actual clue what lodestar have planned):

- transition from node serving `BeaconBlockHeaders` and updates to gossiping this info on p2p network
- implement fork choice rule so that light client can reorg and sync to true head of chain
- serve execution layer data to light clients on Amphora devnet
- spec out how execution data will be served to light clients on post-merge mainnet


## libp2p

It seems sensible to gossip the light client updates rather than rely on a req/resp model. This seems like an ideal task for libP2P's pubsub, because:

- pubsub's interaction model is M:N where M << N. M can be Beacon nodes, N can be light clients
- pubsub expects consumers to subscribe to specific topics. These can be light client updates
- pubsub is an overlay network - it could overlie the existing consensus layer libp2p network
- pubsub is altruism-free, meaning only nodes subscribing to a topic participate in the subnet - this could be the sync committee and light clients
-  


## Outlook for developers

For developers aiming for a Beacon light client the decision to build on top of the existing p2p network or the portal network is not obvious. Developers that want to bootstrap an existing infrastructure to get a light client out the door as quickly as possible, to serve users soon after the merge (or even pre-merge, post-Altair), will be inclined to lean into using benevolent Beacon nodes as their data sources as per Lodestar. This seems to be a sensible stepping stone into serving that same data over the gossip network. Developers with a longer time horizon might be inclined to focus on the portal network. This will necessarily mean engaging in developing the network itself for some significant time before it becomes client development. It seems sensible to have teams working on both...


## links

[portal network specs](https://github.com/ethereum/portal-network-specs)
[LES](https://github.com/ethereum/devp2p/blob/master/caps/les.md)
[lodestar](https://github.com/ChainSafe/lodestar)
[lodestae light client blogpost/video](https://medium.com/chainsafe-systems/a-lodestar-for-ethereum-consensus-1-c2ad6a7b46d9)
