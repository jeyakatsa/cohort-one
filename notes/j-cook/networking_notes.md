
# Networking

## Ethereum networking pre-merge

This doc contains notes on Ethereum's networking layer. The layer can be divided into two stacks: the first is the discovery stack which sits on top of UDP and allows a new node to find peers to connect to and the second is the DevP2P stack that sits on top of TCP-IP and allows nodes to exchange information. These layers act in parallel, with the discovery layer feeding new network participants into the network, and the DevP2P layer enabling their interactions. The DevP2P layer is a broad bit of terminology because DevP2P is really a whole stack of protocols that are implemented by Ethereum to establish and maintain the peer-to-peer network. There seems to be quite a lot of ambiguity in the material available on blogs, videos, slides etc online, so where in doubt I have taken my definitions from my close-read of the Geth source code.

First, why do these two stacks sit on top of different low-level protocols? Dicovery is built on UDP (User Datagram Protocol) whereas the rest of the p2p communication happens over TCP. UDP does not support any error checking, resending of failed packets, or dynamically opening and closing connections - instead it just fires a continuous stream of information at a target, regardless of whether it is successfully received. This minimal functionality also translates into minimal overhead, making this kind of connection very fast. For discovery, where a node simply wants to make its presence known in order to then establish a formal connection with a peer, UDP is sufficient.

However, for the rest of the networking stack, UDP is not fit for purpose. An authenticated, established connection between peers is a prerequisite for Ethereum nodes since this is how the network becomes secure. The informational exchange between nodes is quite complex and therefore needs a more fully featured protocol that can support resending, error checking etc. The additional overhead is worth the additional functionality. Therefore, the majority of the P2P stack operates over TCP.

### Discovery

#### Overview

Discovery is the process of finding other nodes in network. This is bootstrapped using a small set of bootnodes that are [hardcoded](https://github.com/ethereum/go-ethereum/blob/master/params/bootnodes.go) into the client. These bootnodes exist to introduce a new node to a set of peers - this is their sole purpose, they do not participate in normal client tasks like syncing the chain. On receiving a message from a new node, the bootnode returns a list of potential peers that the new node could connect to. Boot nodes only used the very first time a client is spun up - the node table is persisted so subsequent times the client is started up connections are made directly to real peers.

The protocol used for the node-bootnode interactions is a modified form of [Kademlia](https://medium.com/coinmonks/a-brief-overview-of-kademlia-and-its-use-in-various-decentralized-platforms-da08a7f72b8f) which uses a distributed hash table to share lists of nodes. There are currently two versions of this discovery protocol available in Ethereum clients - discv4 and discv5 which modify the Kademlia DHT slightly differently (details in later section). The routing table is a database that stores the information required to find each node in the network (IP addresses, port and ID). Each node has a version of this routing table containing information about its closest peers. The table is divided into "k-buckets" where each bucket represents groups of neighbours that sit at different distances from the node. These are not geographic distances - distance is defined by the XOR metric (i.e. for each bit in a nodeID an XOR returns 0 if two nodes match and 1 if they differ - the sum of  XOR's bits gives the distance). Each node's routing table orders buckets (lists) of neighbours in ascending order of distance, and the node connects to its N closest peers. 

Randomness and regular refreshing of each node's routing table is implemented as a security feature - a random target is generated for each node and its peers are selected based on the distance to that target. The node actually requests the 16 closest nodes to the target, then request the 16 closest nodes from each of those, giving 16 x 16 = 256 potential peers derived from each randomly generated target. The refresh period is a configurable variable in each client, with defaults in various clients ranging from ~5 mins to ~1 day.

In Ethereum, the node ID used in the routing table is actually a [Keccak hash](https://keccak.team/keccak.html) of the node's public key. Other significant differences between Ethereum's protocol and Kademlia include the nuances of the distance calculation (which differs between discv4 and discv5), the process used to determine a node's reputation (which can vary between clients), and the implementation of bonding to protect against DDOS attacks. 

This discovery process uses just 4 messages - 2 requests and 2 responses: 

REQUESTS:

- PING 
- FIND-NODE

RESPONSE:

- PONG
- NEIGHBOUR

These messages are exchanged using [UDP](https://en.wikipedia.org/wiki/User_Datagram_Protocol).

#### Bonding

Discovery starts wih a game of PING-PONG. A successful PING-PONG "bonds" the new node to a boot node. The initial message that alerts a boot node to the existence of a new node entering the network is a PING. This PING includes hashed information about the new node, the boot node and an expiry time-stamp. The boot node receives the PING and returns a PONG containing the PING hash. If the PING and PONG hashes match then the connection between the new node and boot node is verified and they are said to have "bonded". 

Once bonded, the new node can send a FIND-NEIGHBOURS request to the boot node. The data returned by the boot node includes a list of peers that the new node can connect to. If the nodes are not bonded, the FIND-NEIGHBOURS request will fail, so the new node will not be able to enter the network.

Once the new node receives a list of neighbours from the boot node, it begins a PING-PONG exchange with each of them. Successful PING-PONGs bond the new node with its neighbours, enabling message exchange.

```
start client --> connect to boot node --> bond to boot node --> find neighbours --> bond to neighbours
```

So at the end of this stage, the new node has a continuous PING-PONG exchange occurring with a group of peers over UDP. As long as PINGs return PONGs, the peers remain bonded and are ready to chat by establishing a TCP connection.

## Interactions between nodes

After new nodes enter the network, their interactions are governed by protocols in the DevP2P stack. These all sit on top of TCP and include the RLPx transport protocol, wire protocol and several sub-protocols. RLPx is the protocol governing initiating, authenticating and maintaining sessions between nodes. RLPx encodes messages using RLP (Recursive Length Prefix) which is a very space-efficient method of encoding data into a minimal structure for sending between nodes.

A RLPx session between two nodes begins with an initial cryptographic handshake. This involves the node sending an auth message which is then verified by the peer. On successful verification, the peer generates an auth-acknowledgement message to return to the initiator node. This is a key-exchange process that enables the nodes to communicate privately and securely. A successful cryptographic handshake then triggers both nodes to to send a "hello" message to one another. This message is part of the wire protocol. The wire protocol is initiated by a successful exchange of hello messages.

The hello messages contain:

 - protocol version
 - client ID
 - port
 - node ID
 - list of supported sub-protocols
  
This is the information required for a successful interaction because it defines what capabilities are shared between both nodes and configures the communication. There is a process of sub-protocol negotiation where the lists of sub-protocols supported by each node are compared and those that are common to both nodes can be used in the session.

Along with the hello messages, the wire protocol can also send a "disconnect" message that gives warning to a peer that the connection will be closed. The wire protocol also includes PING and PONG messages that are sent periodically to keep a session open. The RLPx and wire protocol exchanges therefore establish the foundations of communication between the nodes, providing the scaffolding for useful information to be exchanged according to a specific sub-protocol. 

This DevP2P layer sits beneath the Ethereum protocol itself, providing the communication infrastructure that Ethereum runs on top of.

### Discv4 and Discv5

Ethereum’s current node discovery protocol is discv4. This is a stripped-down implementation of Kademlia as described above. There is no need for the FINDVALUE or STORE parts of Kadamlia's DHT so it is not used in discv4. In discv4 a new local node bootstraps its way into the network via a bootnode and then requests connections to its closest peers. Distance is not a geographic measure here, it is an encryption distance measured by the number of matchign bits in the encrypted node IDs. This process repeats with successively distant peers until sufficient connections are made. nodes using the discv4 protocol maintain ENR records. This protocol remains the default for Ethereum nodes (at least on geth) but it has some issues that motivate the development of an updated protocol. These issues include missing verification of subprotocols, making it impossible for discv4 discovery to identify peers on other networks such as Ethereum Classic or peers supporting specific subprotocols, at least until a session is already underway. Discv4 also makes use of timestamps which can lead to problems if nodes have inconsistent clocks. Peers are supposed to mutually contribute to discovery ('mutual endpoint verification') but this is not well served in discv4, meaning two peers could disagree on each other's verification status. 

discv5 is an updated discovery protocol. To address the subprotocol issue, topic tables were introduced that can becompared between nodes to find mutually supported subprotocols and can be used to find nodes with specific functionality (e.g. supporting light clients). This is achieved by each node adding its own details to the topic table held by a peer by sending an "ad". This does not scale well to large networks. Endpoint verification has not yet been resolved in discv5. The distance calculation was amended to use the log2 of the XOR'd node IDs. This distance is passed to the FINDNODE function instead of passing an identifier, which returns all the nodes within that distance. Essentially, the process is reversed relative to discv4. However, discv5 has not yet slved

A lot of Geth seems to be dynamically coded so that clients can use v4 OR v5. Beacon clients will use discv5 to discover and connect to peers.


### Sub-protocols

#### Eth (wire)
The core Ethereum transaction protocol that enables sending and receiving blocks, making transactions and related metadata

#### les (light ethereum subprotocol)
minimal protocol for syncng light clients

#### Whisper
Secure messaging between peers without writing any information to the blockchain - many clients do not support this

#### Swarm
Swarm is a decentralised filesystem

Clients that support these sub-protocols exose them via the json-rpc

enode: a url scheme for ethereum nodes: format username@host (nodeID is a hexadecimal # and 
host is IP address). enode can be used to identify bootnodes

### PORTS and Protocols:

8545 TCP: used by http based json-rpc api
8546 TCP: used by Websocket based json rpc api
30303 TCP and UDP: used by P2P network (inbound traffic from other nodes)
30304: UDP used by P2P discovery overlay (outbound traffic from node)

### ENR: Ethereum Node Records

Ethereum node records were introduced as a way to circumvent some of the restrictions imposed by discv4, in particular the inflexibility in node connection information shared between peers during discovery. The Ethereum Node Record (ENR) is an object that contains three basic elements: a signature (hash of record contents made according to some agreed identity scheme), a sequence number that tracks changes to the record, and an arbitrary list of key:value pairs. This list can contain any pairs defined by the user but is expected to contain connection information such as the node's public key, identity scheme used to generate the ENR signature, IP address, ports for UDP, TCP etc. This ENR format is designed to support future upgrades to the discovery network because arbitrary key:value pairs can be added or removed from the record

 the nodes public key (keys, if an execution and consensus client exist on a single network - more later), 

## How is p2p implemented in geth?

There are multiple main.go files - i.e. multiple executables inside geth's cmd directory. These are dev tools for interacting with modules directly, notnecessarily part of the core functionality of geth as it applies to most users. The executable for geth itself is go-ethereum/cmd/geth/main.go. This file is run using `geth` command in the terminal, with many optional flags.

Running `main.go` first checks which flags have been toggled with the `geth` command. Those flags are used to configure a new node instance. The main function inside 'main.go' calls the run method on an app object that is configured inside main.go's `init()` function. There, the CLI is started. The run method calls `geth()`. In that function, several further function calls are made: one to `prepare()` which takes the cli context as a kwarg and uses it to define the network to connect to and configures the cache. Configuring the cache really means defining how much memory is available for the geth client to use - for a full node on mainnet it is set high (4GB), for a light client it is set very low (128 MB). Next, `geth()` calls `makeFullNode()` passing the cli context as a kwarg again. This `makeFullNode` function is in cmd/config.go. It configures the node with information imported from `gethConfig` which is a struct defined in the same file. Two objects are returned - the choice of backend and a variable called stack. Stack comes from a subcall to `node.New()` in enode/p2p/node.go and returns a node struct with allocated memory aliased "stack". That node struct contains the node's enr record and ID (enr explanation coming later).

Having retrieved the backend config and the node struct, these two pieces of info are passed to main.go's `startNode()` in order to start the node that was just configured, open the user wallet and listens to any events emitted from it, and begin downloading the blockchain history until the node is up to date (sync'd). Also, if this is a mining node (as determined by the cli flags) then this function starts the mining process. 

cmd/main.go's `startNode` func takes the cli context (the "state" of the session that includes all the predefined config) and the instantiated node struct. To start up the node itself, the `StartNode()` function from cmd/utils/cmd.go is called with the cli context passed as an argument. This in turn calls `Start()` from node.go passing "stack" which aliases the configured node object.

Inside this node.go `Start()` function, there are several calls out to functions in p2p/server.go. These functions in turn grab the configuration options for discovery and dialing in/out from the server to find and maintain connections with peers. Specifically, in `Start()` the functions `setupListening()`, `setupDiscovery()`, `setupDialShchedule()` and `setupLocalNode()` from server.go are called. server.go `SetupLocalNode()` sets up the crypto handshake that authenticates and secures communications between peers, then sets the IP etc for the local node. `setupListening()` configures the TCP connection the node will use to communicate with peers. `setupDialSchedule()` sets up the outbound messaging from the local node to potential peers.

server.go `setupDiscovery()` is where the specifics of the discovery routine are defined. This is where the list of trustedNodes is populated which is later used to connect the nodes to peers once the server is run. `setupDiscovery` will proceed by starting to listening for messages on the UDP connection then calling either `listenV4()` to implement the discv4 protocol or `listenv5()` to implement discv5, in either case using UDP to ping bootnodes and listen for pongs, then making a `findnode` request and awaiting return a list of peers. There is an additional function `discmix()` that randomizes the peer selection to maintain fairness. 

The final task of the `Start()` function is to spawn a parallel go routine that runs the server. `server.run()` in server.go is where the action is. Here, the P2P networking is actually, finally initiated and the local node makes connection to a bootnode and then real peers as the DHT is populated by the discovery protocol. The node emits messages and listens for responses from peers, which in turn initiates an RLPx session where the nodes can start to ping-pong.

On a successful ping-pong exchange, functions from v4wire or v5wire start an rlpx session over tcp with the newly discovered peers. The function `setupDialScheduler()` that was called in `server.Start()` identifies when the peer table is populated and then starts dialing out to those peers over TCP. Once connected, they swap crypto handshakes to make their exchange secret, then initiate an encrypted RLPx session.



## light clients

Light clients are implementatons of the client software that use minimal resources. The exisiting DevP2P LES network is designed using a client/server architecture – with light-clients as the clients, and full nodes as the servers (today, the term light-client is usually used to refer to a client of the existing DevP2P LES network). Since this architecture puts all the load on full nodes, and full nodes are already heavy to run, most node runners don’t activate this option. So while the current network design serves it’s original purpose well, it is severely lacking from a light-client perspective, primarily because few nodes activate light client support.


## Ethereum networking beyond the merge

The first critical thing to understand is that post-merge, node operators will run two ethereum clients in parallel: an execution client and a consensus client. The execution client is an existing mainnet client (e.g. geth, nethermind...) that will interface with the existing Ethereum network, although with the consensus and block gossip functionality removed. The EVM, validator deposit contract and block production will all still live on the execution layer and be controlled by the execution client. State and history sync and the transaction mempool will also remain in the domain of the execution client. Responsibility for bundling transactions into blocks also stays with the execution client. However, post-merge the relevant function calls for creating and validating blocks will originate at the consensus client. This requires the current Ethereum mainnet's p2p layer to persist beyond the merge to support the execution layer, a local connection between the execution and consensus clients, and a separate consensus p2p layer.

The execution p2p layer will remain the same as it is now, although there is some enthusiasm to deprecate devP2P in favour of libP2P eventually. Certainly, this will come much later than the merge if at all. The execution clients still need to participate in transaction gossip so that they can manage the transaction mempool. This requires encrypted communication between authenticated peers since transactions are valuable. Participation in block gossip is no longer needed, as this is delegated up to the consensus client. There is also no need to engage in PoW validation, as providing valid PoW hashes is no longer relevant for consensus. These execution layer responsibilities necessitate the persistence of both the discovery protocol (including bonding) and the DevP2P layer including RLPx sessions and transactions via the eth sub-protocol. Encoding will remain predominantly RLP.

The execution layer now also needs to receive instructions from, and return data to, the consensus client. Best practice is for a validator to run both the execution and consensus clients locally, as outsourcing the execution layer to a third party opens a significant attack surface (e.g. a malicious execution client operator could generate invalid transactions to force a slashing event, or simply retain all the block rewards). This means that communication between the two clients can be achieved using a local RPC connection. Since both clients sit behind a single network identity, they share a ENR (Ethereum node record) which contains a sepaate key for each client (eth1 key and eth2 key).

The consensus client needs to participate in block gossip in order to receive blocks to validate and broadcast new blocks when it acts as block proposer. This requires a discovery layer that connects the consensus client to other trusted consensus clients on the network, and a TCP-based P2P layer for sending Beacon blocks, attestations etc in secure sessions. For the consensus layer, the p2p layer has been built using libP2P rather than devP2P. The discovery protocol uses discv5 over UDP. RLP encoding is replaced by SSZ (simple serialize) and snappy compression in the consensus layer. 

To validate transactions, the consensus layer has to pass an execution payload to the execution layer. This payoad contains the transactions which is executed by the EVM and the results returned to the consensus layer. The control flow is as follows, with the relevant networking layer shown in brackets:

When consensus client acts as validator:
 - consensus client receives a block via the block gossip protocol (consensus p2p)
 - consensus client prevalidates the block, i.e. ensures it arrived from a valid sender with correct metadata
 - The transactions in the block are sent to the execution layer as an execution payload (local RPC connection)
 - The execution layer executes the transactions and validates the state in the block header (i.e. checks hashes match)
 - execution layer passes validation data back to consensus layer, block now considered to be validated (local RPC connection)
 - consensus layer now adds block to head of chain and attests to it, broadcasting the attestation over the network (consensus p2p)

When consensus client is block producer
- consensus client receives notice that it is the next block producer (consensus p2p)
- consensus layer calls `create block` method in execution client (local RPC)
- execution layer accesses the transaction mempool which has been populated by the transaction gossip protocol (execution p2p)
- execution client bundles transactions into a block, executes the transactions and generates a block hash
- consensus client grabs the transactions and block hash from the consensus client and adds them to the beacon block (local RPC)
- consensus client broadcasts the block over the block gossip protocol (consensus p2p)
- other clients receive the proposed block via the block gossip protocol and validate as described above (consensus p2p)

Once the block has been attested by sufficient validators it is added to the head of the chain, justified and eventually finalised. The difference between justification and finalisation is that although they have passed initial validation and are intended to become part of the canonical chain history, it is still possible to roll back justified blocks. Finalised blocks are an immutable part of the historical chain shared by the entire network.

<br></br>
<img src="./assets/cons_client_net_layer.png" width=500>
<br></br>
<img src="./assets/exe_client_net_layer.png" width=500>
 
Network layer schematic for post-merge consensus and execution clients, from [ethresear.ch](https://ethresear.ch/t/eth1-eth2-client-relationship/7248)
<br></br>


## What are the REQ/RESP, topics and discovery domains?

The networking layer for the consensus layer can be split into three domains. Discovery has been covered in some detail already in this document but to recap it will use the discv5 protocol as per many of the existing Ethereum mainnet nodes. This runs over UDP and supports ENR lookup. Importantly, this includes topic advertisements used for protcol negotiations. A major difference between the existing nodes' implementation of discv5 and that of the consensus layer is that the latter implements an adaptor to integrate discv5 into a libP2P stack, deprecating devP2P. The reasons for this are nuanced, but essentially a networking stack built with libP2P eases the transition from discovering new peers to establishing connections and streams with them.

The ENR for consensus nodes includes the node's public key, IP address, UDP and TCP ports and two consensus-specific fields: the attestation subnet bitfield and eth2 key. The former makes it easier for nodes to find peers participating in specific attestation gossip sub-networks. The eth2 key contans information about which Ethereum fork version the node is using, ensuring peers are connecting on the right Ethereum. These ENR records are therefore required in discovery.

After discovery, all communications happen on the TCP transport layer which is a libP2P stack. Clients can dial and listen on IPv4 and/or IPv6 as defined in their ENR. RLPx is deprecated in favour of libP2P's noise secure channel handshake with secp256k1 identities. This defines the transport layer on top of which the gossip and req/resp protocols operate.

The gossip domain includes all information that has to spread rapidly throughout the network. This includes beacon blocks, proofs, attestations, exits and slashings. This is transmitted using libP2P gossipsub v1 and relies on various metadata being stored locally at each node, including maximum size of gossip payloads to receive an transmit. 

Beacon blocks and aggregation/proofs are considered primary global topics to be propagated to every node on the network. 




## Why does the consensus client prefer SSZ to RLP?

SSZ stands for simple serialization. It uses fixed offsets that make it easy to decode individual parts of an encoded message without having to decode the entire structure, which is very useful for the consensus client as it can efficiently grab specific pieces of information from encoded messages. It is also designed specifically to integrate with Merkle protocols, with related efficiency gains for Merkleisation. Since all hashes in the cons-client are Merkle roots, this adds up to a significant improvement. SSZ also guarantees unique representations of values.

## Altair

Altair adds new messages, topics and data to the Req-Resp, Gossip and Discovery domain. Some Phase 0 features will be deprecated, but not removed immediately.

validators sign blocks, light client trusts validators to be honest
light client needs head of chain and state information
headers contain state root
also requires functionality to add transactions to execution layer tx mempool (or eventually another shard) or via an L2/dapp

## Links:
networking video https://www.youtube.com/watch?v=hnw59hmk6rk&list=PLNLh1EyDzSGP-lkNCBhCptoJ-NMu_BYfS&index=3
kademlia to discv5 https://vac.dev/kademlia-to-discv5
eclipse attack paper (contains nice explanation of Geth p2p in 2018) https://eprint.iacr.org/2018/236.pdf
kademlia paper https://pdos.csail.mit.edu/~petar/papers/maymounkov-kademlia-lncs.pdf
intro to ethereum p2p video https://p2p.paris/en/talks/intro-ethereum-networking/
rlpx specs https://github.com/ethereum/devp2p/blob/master/rlpx.md
scope of the merge http://ethresear.ch/t/the-scope-of-eth1-eth2-merger/7362
eth1/eth2 relationship http://ethresear.ch/t/eth1-eth2-client-relationship/7248
merge and eth2 client details video https://www.youtube.com/watch?v=zNIrIninMgg




## scratch notes 
Altair - beacon contains comittee sig


light client path not availble in prysm (available in lodestar though)
portal network designed for eth1 primarily
what network requirement for retrieving relevant data - pubkeys, next committee
header for validating state, also do something w state - balances, validators, staking balances etc
how is validator using state to validate?
eth2 - discv5, libp2p pubsub, req/res from libp2p
portal network uses parts not all - used v5
q: json-rpc requires exe-client?
does a beacon portal network necessarily need to also be/interface with an execution layer portal network to enable transactions?


- Q1: Does the consensus client expose its own json-rpc? I guess not, unless interactive access to Beacon state is needed?
- Q2: Will all 64 shards be executable? Will they all eventually support evm?
- Q3: Would a Beacon light client need a p2p connection to the execution layer in order to submit transactions?
    - this ties in to Q1 as a light client needs access to the json-rpc endpoint, but this is exposed by the execution client not the consensus client
    - Q3.5: does this imply the beacon light client p2p should overlay/extend the existing portal network rather than standing alone?
- Q4: Why does the portal network favour uTP as its base-level protocol instead of e.g. UDP?




