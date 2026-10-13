# Socket P2P — Peer-to-Peer Network Protocol Spec (archived)

> **Archival reference.** This repository preserves the **P2P Guide** and **P2P Spec**
> from the Socket Supply *Socket Runtime* documentation, captured from the Internet Archive
> on **2024-12-19**. The Socket Supply project is no longer actively maintained; this copy is
> kept for reference and study.
>
> - **Source:** <https://web.archive.org/web/20241219205807/https://socketsupply.co/guides/#p2p-spec>
> - **Original author:** Socket Supply Co. — Paolo Fragomeni and contributors
> - **Status (as published):** `DRAFT`
>
> The spec text below is reproduced from the original documentation. The packet/peer
> structures, constants, and the `STATE_0`-`STATE_6` protocol state machine are preserved as
> published; only the formatting was converted from HTML to Markdown. Wording is unchanged.
> All rights to the original content remain with its authors.

## What this is

Socket Supply's P2P transport (shipped in the Socket Runtime as the `socket:network` module)
does **NAT traversal over UDP with little or no infrastructure** - no STUN, no TURN, no signalling
servers - and stays live as peers come and go. It solves only the **Network** domain of P2P
(discovery + packet exchange), leaving storage, consensus and compute to the application.

This document is the protocol's own guide and specification, in two parts:

1. **P2P Guide** - the why, plus how NAT traversal, reflection and hole-punching work.
2. **P2P Spec** - the packet and peer data structures, constants, and the formal
   state machine (`STATE_0` Initial -> `STATE_6` Exchange).

---

## Related specifications (from `nat-traversal-spec`)

Alongside the narrative guide below, this repo also preserves the **more formal
NAT-traversal specification** and its **network simulator**, archived from the separate
`socketsupply/nat-traversal-spec` repository:

- **[`SPEC.md`](./SPEC.md)** — the formal, implementer-facing NAT-traversal spec (RFC-style
  `SHOULD`/`MUST`), with concrete constants (`LOCAL_PORT 3456`, `TEST_PORT 3457`, `BDP 10`,
  `BDP_MAX_PACKETS 1000`, `KEEP_ALIVE_TIMEOUT 29_000`), data structures, and the full message
  set (`MsgPing`, `MsgPong`, `MsgIntro`, `MsgLocal`, `MsgJoin`, `MsgRelay`, `MsgConnect`,
  `MsgTest`). This is a **different, more detailed version** than the "P2P Spec" section of the
  guide below (which uses `STATE_0`–`STATE_6`); keep both for the fuller picture.
- **[`SIMULATOR.md`](./SIMULATOR.md)** — the deterministic network-simulator design used to test
  the protocol (event queue, random message ordering, seeded replay).

> Note: the upstream repo also had a `tla/index.tla` (TLA+ model) referenced by its README, but
> that file is an **empty placeholder** upstream (0 bytes), so there was nothing to copy.

## P2P Guide

Status: DRAFT

### Overview

P2P is an approach to building software that allows its users to communicate directly. It offers increased privacy, security, and performance, as well as reduced cost and complexity when compared to more traditional and less transparent Cloud infrastructure. It's relevant to anyone building multi-user software on a scale of small to large (millions of users).

P2P is also a broad term that can mean a lot of things to a lot of different people. To clarify the conversation, we separate P2P info from 4 domains. `Network`, `Storage`, `Consensus`, and `Compute`.

Our solution focuses exclusively on the Network. Our goal is to make sure you can discover other computers and exchange packets with them in the most reliable way while having the least impact on your application or its logic.

### Criteria

- Do not use more battery or storage than any other app.
- Ensure continuity of the overall network regardless of the condition of any other individual network resource.
- Ensure communication regardless of the state of any well-known peers.

### Goals

A front-end developer familiar with HTML, CSS, and JavaScript should be able to create a fully functional chat app like Telegram, a social media app like Twitter, a coordination app like Uber, or a collaborative content creation app like Figma or Notion without needing any servers or Cloud services. Here's the way we see these goals.

#### Complexity

Servers are natural bottlenecks (One server to many users), and scaling them up quickly becomes a complex distributed system of shared state. A P2P network is many users to many users, and although it is also an eventually consistent, distributed system of shared state, the total complexity of all components needed to create a robust P2P network is finite, and transparent. And unlike Cloud services, since it's entirely transparent, it can be verified with formal methods by anyone.

#### Security & Sovereignty

P2P is simply a networking technique, it's not more or less dangerous than any other approach to computer networking. A P2P app is not more likely to contain malicious code or perform unwanted behavior than any web page or client-server application. P2P is entirely transparent which makes it easier to audit.

Conversely, Cloud services are a closed system — owned and operated by a private third party. Most people can't audit them. With Gov. Cloud, auditing is intermittent. There is always a non-zero chance for a greater number of incidents than if every single bit was self-contained. Fewer moving parts means less risk.

#### Cost

As your application grows, so do the costs of the services you use. Growth usually means combining services from different providers and staffing the experts to glue it all together.

### How P2P Works

**Client-Server:** Many users communicate by sending all their data to one end-point; typically rented by a company from a Cloud hosting provider.

**Peer-To-Peer:**

Many users communicate in coordination with each other, sharing only with each other, or in concert with existing services.

```
import { network, Encryption } from 'socket:network'

//
// Create (or read from storage) a peer ID and a key-pair for signing.
//
const peerId = await Encryption.createId()
const signingKeys = await Encryption.createKeyPair()

//
// Create (or read from storage) a clusterID and a symmetrical/shared key.
//
const clusterId = await Encryption.createClusterId('TEST')
const sharedKey = await Encryption.createSharedKey('TEST')

//
// Create a socket that can send a receive network events. The clusterId
// helps to find other peers and be found by other peers.
//
const socket = await network({ peerId, clusterId, signingKeys })

//
// Create a subcluster (a partition within your cluster)
//
const cats = await socket.subcluster({ sharedKey })

//
// A published message on this subcluster has arrived!
//
cats.on('mew', value => console.log(value))

//
// A message will be published into this subcluster
//
cats.emit('mew', { food: true })

//
// Another peer from this subcluster has directly connected to you.
//
cats.on('#join', peer => {
  peer.on('mew', value => {
    console.log(value)
  })
})

```

#### IP Addresses & Ports

To get connected to the Internet, you get an account from an Internet Service Provider (an ISP). For home or office Internet, your ISP gives you a router that you plug into the wall. Once you have that, you connect your various devices to it. On mobile, you also connect to a router, but you never see it. Every device connects to the Internet through a router.

Everything that connects to the Internet needs an address. The purpose of the address is to help other computers know where to deliver messages (packets).

Your router is assigned an IP address by your ISP. Your computer is assigned a local IP address by your router. In fact, any computer you connect to your router will get a unique local IP address. All of these addresses can change at any time.

But IP addresses are not enough information to start communicating. Imagine your computer is like an office building, its IP address is like the street address, and every program that runs is like a different office in that building. Each office gets assigned a unique number which we call the "internal port".

#### Routers & NATs

Now imagine lots of programs running on lots of computers all want to send packets. To manage this, the router maintains a database that maps a program's internal port and the IP address of the computer that it's running on to a unique external port number. This is done so the router can ensure that inbound packets always reach the program running on the computer that is expecting them. This is called Network Address Translation, or NAT for short. And different routers made by different manufacturers for different purposes can have different behaviors!

Your computer’s local IP address isn't visible to the outside world, and the router’s IP address isn’t visible to your computer, and neither are the port numbers in the router's database. To make things even more difficult, if the router sees inbound packets that don't match an outbound request, it will usually discard them. Different routers may even assign port numbers differently! All this makes it hard to get server-like behavior from your mobile device or laptop. But with P2P, we want to listen for new packets from people we don't know, kind of like a server. This is where it starts to get complicated.

#### Reflection

Before other programs on the Internet can send packets directly to a program on your computer or device, they need to know your router's public IP address, and the external port number that your router assigned to your program. They also need to know what kind of NAT behavior to expect from your router.

This is determined by asking for our address and port info. It's important to ask two other peers (and they must be outside of our NAT). If both peers respond with the same port, we are behind an Easy NAT. If both respond with different ports we are behind a Hard NAT. If we respond to an unsolicited query on a well- known port, we are behind a Static NAT. We call this process of asking another peer for address information "Reflection".    Step 1

Reflection is where Alice asks two other peers for her external IP address and port. Those two other peers, `X` and `Y`, can be any arbitrary Static or Easy peers, but they must be outside of Alice's NAT.    Step 2

If Alice wants to talk to Bob, they both tell Cat their information. Cat will send Alice Bob's address, port, and NAT type. Cat will also send Bob Alice's address and port and NAT type. Now they have been introduced.

A peer that can introduce us to other peers is called an "introducer". An introducer can be an iPhone, a laptop, or an EC2 instance, it could be anything as long as it has an Easy or Static NAT type.

#### NAT Traversal

Now that Alice and Bob know each other's information, are ready to start the process of initiating a "connection". We use UDP which is message based, so this is only a connection in the sense that each peer knows about the other's address information are they are expecting messages from each other. This process is called NAT traversal (aka "Hole Punching"). Alice and Bob's NAT types will determine how well this initiation will go.

If you've ever set up a Cloud VPS (like Amazon EC2), you've indirectly configured its router to allow direct traffic on a particular port. This is an example of a Static NAT. A peer with this kind of NAT will accept packets without any extra work.

If Alice and Bob are both on Easy NATs, that means they are probably on phones or home networks. The procedure is pretty simple. Alice must first send a packet to Bob. This packet will fail to be delivered but will open a port on her router. Bob does the same thing.    Step 3

At this stage, the packets being sent are never received, they are only meant to open the port on the router that is sending them.    Step 4

The router will maintain a port mapping in its database for `±30 seconds`. Now is the time to start sending heartbeat packets so the port mappings don't expire.

Now that a port is open and the router is expecting to see responses addressed to it, Alice and Bob can send messages and their routers will not consider the messages unsolicited. But if either Alice OR Bob are on a Hard NAT, the process is similar but using more ports and sending more control messages.

If Alice is on the Hard NAT, she opens 256 ports, and Bob immediately sends packets until he receives a packet from Alice confirming that at least one of his packets was received.

This generally works better than it sounds due to probability. For example, if you survey a random group of just 23 people there is actually about a 50–50 chance that two of them will have the same birthday. This is known as the birthday paradox, and it speeds up this guessing progress so that connection times in this scenario are under a second, once the port is open it can be kept open. And only about a 1/3rd of all NATs are Hard, so connection times are about the same as they are in client-server architectures. Also, this whole process doesn't work if both NATs are Hard.
>

Note: There is an optimization where you can check if the router supports port mapping protocols such as PmP or PnP, but in our research, very few routers respond to queries for these protocols.

Now you have a direct connection and you can try to keep it alive if you want. But it will likely go offline soon, most modern networks are extremely dynamic. Imagine taking your phone out to check Twitter for a few seconds, then putting it back in your pocket. So the next part of our protocol is equally as important as the NAT traversal part.

#### Disruption & Delay Tolerance

In modern P2P networks, all peers should be considered equally unreliable. They may be online in short bursts. They may be online but unreachable. They may be offline for a few seconds, or offline for a month. This protocol anticipates these conditions in its design — any node, or any number of nodes are able to fail and this will not affect the integrity of the overall network.

Let's say that Alice and Bob get connected and want to have a video chat. They can send packets of data directly to each other. They may have minor network interruptions that cause inconsequential packet loss. But what if Alice and Bob are text messaging? A lot of the time this kind of communication is indirect (or asynchronous). If Bob goes offline, how can he receive the subsequent messages that Alice sends? This is where we need to dig into the protocol design.

#### Protocol Design

Stream Relay is a replicating, message-based gossip protocol, that follows the criteria for Disruption Tolerant Networks.

In the simplest case, Epidemic routing in gossip protocols deliberately makes no attempt to eliminate flooding. However, protocols such as MaxProp (John Burgess Brian Gallagher David Jensen Brian Neil Levine Dept. of Computer Science, Univ. of Massachusetts, Amherst), demonstrate optimizations that can outperform protocols with access to oracles (An oracle being something like a Distributed Hash Table, a Minimum Spanning Tree, or a Broadcast Tree).

A replicating protocol casts a wide net and yields faster response times and higher hit rates, but in some cases there are arguments it will waste more bandwidth.

In the paper "Epidemic Broadcast Trees" (Plumtree, Joao Leitao, et. al 2007), the authors define a metric (Relative Message Redundancy), to measure the message overhead in gossip/replicating protocols. However the paper was published before the dominance of mobile devices and their usage patterns. It advocated for the conservation of bandwidth using a "lazy push approach", which as a trade-off, made it slower. It also introduced central services for calculating trees to optimize routing. With mobile usage patterns, there is a smaller window of time to satisfy a user request. So this optimization is no longer relevant, especially if we factor the declining cost of bandwidth and the increasing expectations for responses.

Stream Relay adopts many of MaxProp's optimizations in decisions about packet delivery and peer selection, including optimizations where peers are weighted on their temporal distance (the delta of the time between when a message was sent and a response was received).

Stream Relay's atomic unit of data is a re-framed UDP packet. A key concept is that packets have IDs that are a sha-256 hash of their content and potentially a previous packet ID. Packets are encrypted and causally linked so that they can be delivered in any order by any peer and the recipient can become eventually consistent even under frequently changing or adversarial network conditions.

In simpler terms, this means Alice can send messages to Bob, even if Bob goes offline. Messages can persist in the network, moving from peer to peer until they run out of hops or they are expired and evicted from a peer's cache.

All network packets are encrypted and unless you have the key to decrypt the packet you will never know what's in it. This is critical, because a packet may be buffered and relayed by a peer who is not related to your cluster.

Consider for example all the other peers in your cluster are offline, but you want to send a packet to them, and the packet will be buffered into the network — when the other peers come back online they can receive the packet since it is causally linked.
>

In the figure above, a peer (blue solid line) from Application A wants to send a packet but all other peers are offline, the packet will be buffered to other peers who are members of completely unrelated clusters. This is not the exact route that packets will take, but it roughly illustrates how some copies of the packet will become lost or redundant. The dotted black lines represent queries by the peers who rejoin the network and become eventually consistent.

When you publish to the network, 3 peers are selected. The peer selection process is weighed on the temporal distance and availability of peers (the delta of the time between when a message was sent and a response was received, as well as the average uptime of a peer). The process of picking which messages to send is based on what is directly requested, but also which packets have the lowest hop count.

Each of the selected peers in turn will cache the packet for a 6-hour TTL and replicate the packet increasing the packet's hop count until it reaches 16 at which point it is dropped. When the packet expires, it is rebroadcast one last time to 3 random peers in the network. This approach is effective for reaching the majority of intended peers, and in the case a packet isn't delivered, the recipient only needs 1 of N packets to query the network for a missing packet.

This approach is considered to be a Buffer Scheme Optimization, and is quite well-researched in DTNs.

#### Protocol Performance

Both P2P and Cloud platforms are distributed systems that rely on eventual consistency for most of what they do. But how we measure performance in both is very different, mostly because of their network topology.

In client-server, metrics like P99 help measure the performance of a cloud service behind a single endpoint. Let’s say you have 100 requests; you expect 99 responses with less than (or equal to) a specified latency.

Metrics like these work in a many-to-one model (many clients to a single endpoint).

But P2P networks are a many-to-many model. Peers also have unpredictable latency, they may be online infrequently or for unpredictable durations.

In the topology we're describing, we measure how long it takes for a cluster to become eventually consistent. Or how long it takes for a query packet to find a particular packet. Trace-route packets provide some insight as to how information is flowing through the network.

Here the Y axis is network size (`±150 peers`), and the X axis is time (a `15-minute` sample). Each peer has an average lifetime of `±24 seconds`. Peers joining the network are represented by a green line. And peers leaving the network are represented by a red line. In this sample set we see a total network change of `±39900.00%` — `100%` being the initial size of the network).

This chart tracks a single packet that reaches `±91.84%` of the subscribed peers in less than `<0.72 seconds` (before any queries are needed). A solid black line represents the peers that have received the packet, while the dotted black line above it represents the total number of peers that want to receive it.
>

Note: as the global network grows, response times improve, packets can live in the network longer, and increased packet volume is offset by the increased volume of peers.

#### Protocol Cost

The average cost distribution to each peer in the network is `±0.000205Mb`, with a message redundancy of `±0.017787%`. This is a nominal cost compared to the cost of the average Web page (results varied widely, these are averages over `±50 runs`). As with the Plumtree paper, control packets are not factored into the total network cost.

|  Site |  Initial Pageload |  1st 15s of clicks/scrolling |
|  discord.com |  ± 28Mb |  ± 5Mb (and climbing) |
|  twitter.com |  ± 9Mb |  ± 18.5Mb (and climbing) |
|  google.com |  ± 4.5-6Mb |  ± 70Mb (every click reloads content) |
|  yahoo.com |  ± 36Mb |  ± 80+Mb (and climbing) |

#### Why UDP?

Most modern protocols, like HTTP3 are built on top of UDP.

TCP was historically considered an ideal choice for packet delivery since it's considered "reliable". With TCP packet loss, all packets are withheld until all packets are received, this can be a delay of up to 1s (as per RFC6298 section 2.4). If the packet can't be retransmitted, an exponential backoff could lead to another 600ms of delay needed for retransmission.

In fact, `Head-of-Line Blocking` is generally a problem with any ordered stream, TCP (or UDP with additional higher-level protocol code for solving this problem).

This is undesirably in an extremely volatile network topology where packets will never be sent because of peers leaving the network suddenly and frequently.

UDP is only considered "unreliable" in the way that packets are not guaranteed to be delivered. However, UDP is ideal for P2P networks because it’s message oriented and connectionless (ideal for NAT traversal). Also because of its message-oriented nature, it's light-weight in terms of resource allocation. It's the responsibility of a higher-level protocol to implement a strategy for ensuring UDP packets are delivered.

Stream Relay Protocol eliminates Head-of-Line blocking entirely by reframing packets as content-addressable Doubly-Linked lists, allowing packets to be delivered out of order and become eventually consistent. Causal ordering is made possible by traversing the previous ID or next ID to determine if there were packets that came before or after one that is known.

In the case where there is loss (or simply missing data), the receiver MAY decide to request the packet. If the peer becomes unavailable, query the network for the missing packet.

The trade-off is more data is required to re-frame the packet. The average MTU for a UDP packet is ~1500 bytes. Stream Relay Protocol uses ~134 bytes for framing, allocating `1024` bytes of application or protocol data, which is more than enough.

Read more about disruption tolerant networks.

### Getting Started with P2P

Before you start, there are two types of communication to consider. Indirect, and direct.

#### Streaming

When two peers are online, it's easy to connect them and stream an unlimited amount of data between them. This mode is ideal for things like audio and video.

```
import { network, Encryption } from 'socket:network'

const peerId = await Encryption.createId()
const signingKeys = await Encryption.createKeyPair()

const clusterId = await Encryption.createClusterId('TEST')
const sharedKey = await Encryption.createSharedKey('TEST')

const socket = await network({ peerId, clusterId, signingKeys })

const cats = await socket.subcluster({ sharedKey })

cats.on('#join', peer => {
  peer.on('mew', value => {
    setTimeout(() => {
      cats.emit('mew', videoFrameBuffer)
    }, 1024)
  })
})

```

#### Replicated

Not everyone is always online at the same time! In this case, your message will be encrypted, broken up into little pieces called packets and published to random peers in the network. There will be a selection bias for peers in your cluster. So it's most likely that people who you know will help buffer this data. Packets are not only encrypted but also causally linked, which means they can be received by any peer in any order and you will become eventually consistent with your subcluster.

```
import { network, Encryption } from 'socket:network'

const peerId = await Encryption.createId()
const signingKeys = await Encryption.createKeyPair()

const clusterId = await Encryption.createClusterId('TEST')
const sharedKey = await Encryption.createSharedKey('TEST')

const socket = await network({ peerId, clusterId, signingKeys })

const cats = await socket.subcluster({ sharedKey })

const buf = Buffer.from(JSON.stringify({ volume: 1 }))
cats.emit('mew', buf)

cats.on('mew', buf => {
  const value = JSON.parse(buf)
  console.log(value.volume)
})

```

### Further Reading

See the draft specification or the alpha release for more details.

## P2P Spec

Status: DRAFT

This section describes the intended `behavior` of `program` execution. `behavior` is defined as a sequence of `states`. A `state` is the assignment of values to variables. A `program` is modeled by a set of behaviors: the behaviors representing all material executions.

### Data

#### Packet

```
enum NAT {
  Unknown
  Static
  Easy
  Hard
};

struct packet {
  unit8 type; // 1 bytes
  uint8 version; // 1 bytes
  uint32 hops; // 4 bytes
  uint32 clock; // 4 bytes
  uint32 index; // 4 bytes
  char[32] cluster; // 32 bytes
  char[32] previousId; // 32 bytes
  char[32] nextId; // 32 bytes
  char[32] to; // 32 bytes
  char[1024] message; // 1024 bytes
};

┌──────┬─────────┬──────┬───────┬───────┬─────────┬──────────┬──────┬─────┐
│ TYPE │ VERSION │ HOPS │ CLOCK │ INDEX │ CLUSTER │ PREVIOUS │ NEXT │ TO  │
│ 1b   │ 1b      │ 4b   │ 4b    │ 4b    │ 32b     │ 32b      │ 32b  │ 32b │
├──────┴─────────┴──────┴───────┴───────┴─────────┴──────────┴──────┴─────┤
│ MESSAGE BODY                                                            │
│ 1024b                                                                   │
└─────────────────────────────────────────────────────────────────────────┘

```

The `previousId` is used for causal ordering and MAY be the `packetId` of a previous packet. Causality is NOT history (happened before). The `clock` property is a logical clock that indicates historical ordering. The `nextId` will only exist in cases where the packet is fragmented so that it can meet the requirements of the MTU.

```
                      ┌──────────────>─────┐     ┌┈ ┈
                ┌─────┼──<────┐            │     │
┌──────┬─────┬──┴─┬───┴──┐ ┌──┴───┬─────┬──┴─┬───┴──┐
│ PREV │ ... │ ID │ NEXT │ │ PREV │ ... │ ID │ NEXT │
├──────┴─────┴────┴──────┤ ├──────┴─────┴────┴──────┤ ...
│ MESSAGE                │ │ MESSAGE                │
└────────────────────────┘ └────────────────────────┘

```

#### Peer

```
struct Peer {
  char[32] address; // the ip address of this peer
  char[32] port; // the numeric port of this peer
  string peerId; // the unique id of the peer, 32 bytes
  NAT natType; // the type of the nat in front of this peer
  string clusterId; // the cluster id of this peer
  string pingId; // a string (the last pingId received from this peer)
  time_t lastUpdate; // unix timestamp when this peer was updated
  time_t lastRequest; // unix timestamp when a request was made to this peer
  time_t distance; // the delta between .lastUpdate and .lastRequest
  uint64 geospacial; // the 64-bit unsigned int that represents this peer geolocation
};

```

#### Constants
|  Name |  Description |
| --- | --- |
|  `KEEP_ALIVE` |  the average time that a NAT retains a port mapping, initially `30_000` |
|  `PEERS` |  a list of `Peer` to connect to (read from disk or hard-coded) |
|  `CLUSTERS` |  a list of cluster ids for a peer to subscribe to (read from disk or hard-coded) |

### States

All states apply to all peers, a peer MUST not have unique properties or states in order to maintain network symmetry.

#### STATE_0 (Initial)

##### 0. Description

Initially, `Peer A` may not be addressable directly (because of its NAT). However, the details needed to address it can be discovered by coordinating with other peers located outside its NAT.

Coordination starts when peer `A` sends a packet to peer `B` and peer `C`. `B` responds to `A` with `A`'s public-facing `IP_ADDRESS` and `PORT`. If `C` also responds with the same address, and port, `A`'s `NAT` is `Easy`. If the port number is different, `A`'s `NAT` is `Hard`. If `C` can send a packet directly to `A` on a well-known port, `A`'s `NAT` is `Static`.

Here's an analogy. You don't know what you look like, but a mirror can tell you. If you look in a second mirror and see the same thing, you know the first mirror is correct.

##### 0. Topology

##### 0. Procedure

- Start an interval-function repeating at `KEEP_ALIVE`

- Request this peer's `NAT`, external `IP_ADDRESS` and `PORT`

- Select 2 random peers from `PEERS` with a `NAT` that is `Easy` or `Static`
- Send each peer `Packet<Ping>`

- Each peer that receives the packet will enter `STATE_1`
- `STATE_2` must be completed before the peer can enter a higher state
- AFTER `STATE_2` has been completed

- For each known `Peer`

- IF `now` - `peer.lastUpdate` > `KEEP_ALIVE`, delete the peer
- ELSE GOTO `STATE_1`

- For each cached `Packet`

- IF the packet has expired (`packet.timestamp` > `now` - `packet.ttl`)

- Decrement `packet.postage` (which starts at `8`) by one
- IF `packet.postage > 0`

- multicast to at least 3 peers

- ELSE delete the packet from the cache

#### STATE_1 (Ping)

##### 1. Description

When a peer sends a `Packet<Ping>` and it is received, the Ping sequence is started. This data structure extends the `Packet` struct.

```
struct Ping : Packet {
  uint8 type = 1;
  struct message {
    bool isReflection = false; // optional
    bool isConnection = false; // optional
    uint8 natType; // optional
    SHA256 peerId;
    SHA256 requestId;
    uint64 geospatial;
    uint16 testPort; // optional
  }
}

```

##### 1. Topology

##### 1. Procedure

- LET `PORT` be the remote port
- LET `IP_ADDRESS` be the remote address
- LET `p` be the `Packet<Ping>` received
- IF `p.message.isReflection` is true, send `Packet<Pong>` to `IP_ADDRESS` on `PORT`

- IF `p.message.testPort` is a valid port

- Send `p` to `IP_ADDRESS` using `testPort` instead of `PORT`

#### STATE_2 (Pong)

##### 2. Description

AFTER a peer sends `Packet<Ping>`, it may or may not receive a `Packet<Pong>`. This data structure extends the `Packet` struct.

```
struct Pong : Packet {
  uint8 type = 2;
  struct message {
    char[32] peerId;
    uint64 geospacial;
    uint8 port;
    uint8 testPort;
    char[17] address;
    uint32 cacheSize;
    uint8 natType;
    bool isReflection;
    bool isConnection;
    string pingId;
  }
}

```

##### 2. Topology

##### 2. Procedure

- LET `p` be the `Packet<Pong>` received by this peer.
- IF `p.message.isConnection` is `true`, the packet is a response to an attempt to initialize a connection.

- SET this peer's `.lastUpdate` property to the current unix timestamp
- find the peer in this peer's list of peers and SET its `.pingId` to `p.message.pingId`, this will be observed by any interval-functions that have been started in `STATE_3`.

- IF `p.message.testPort` was provided

- SET this peer's NAT type to `Static`

- ELSE IF `p.message.pingId` was provided AND this peer's `.reflectionId property` is null

- SET this peer's `.reflectionId` to `p.message.pingId` so that it can be checked the next time this state is entered.
- SET this peer's NAT type to `Easy`

- ELSE IF `p.message.pingId` was provided and it is equal to this peer's `.reflectionId`

- IF `p.message.port` is equal to this peer's port

- SET this peer's NAT type to `Easy`

- ELSE SET this peer's NAT type to `Hard`

- send a `Packets<Ping>` with `.heartbeat = true` to the peer that sent `p`.

#### STATE_3 (Intro)

##### 3. Description

In `STATE_5` the two peers are "introduced", meaning they now both know each other's external IP address and port. The next part of the process is known as hole punching. The peer doing the introduction will send a `Packet<Intro>` to `A` with the public `IP_ADDRESS` and `PORT` of `B`, and at the same time send `B` the public `IP_ADDRESS` and `PORT` of `A`.

The first ping does not expect a response. It is only sent to open a port.
>

NOTE Support for port mapping protocols is scheduled for version 2 of this protocol.

##### 3. Topology

##### 3. Procedure

- LET `p` be the received `Packet<Intro>`
- IF this `NAT` is `Easy` AND `p.message.natType` is `Easy` OR `Static`

- send `Packet<Ping>` to the peer at `p.message.address` on `p.message.port`

- ELSE IF this `NAT` is `Easy` AND `p.message.natType` is `Hard`, apply BDP strategy ("Easy" side)

- Send `nPacket<Ping>` messages from our main port to unique random ports at `p.message.address`. On average it will take about `+/- 250` packets. Sending only `250` packets would mean that the connection succeeds 50% of the time, so try at least `1000` packets, which will have a 97% success rate. Give up after that in case the other peer was actually down or didn't try to connect. These packets should be sent at a short interval (ie, `10ms`). This gives time for response messages. If a response is received then stop.

- ELSE IF this `NAT` is `Hard` AND `p.message.natType` is `Easy`, apply BDP strategy ("Hard" side)

- Send `256Packet<Ping>` from unique random ports to `p.message.port`. This means binding `256` ports. The peer may reuse the same ports for subsequent requests. These packets are necessary to open the firewall for the Easy side, and should be sent immediately.

- `.peerId` MUST be set to this `.id`
- `.natType` MUST be set to this `.nat`

- ELSE IF this `NAT` is `Hard` AND `p.message.natType` is `Hard`

- Send `Packet<Intro>` to other peers who have a `NAT` that is NOT `Hard`

#### STATE_4 (Publish)

##### 4. Description

Peers may communicate directly or indirectly. This is a VERY important distinction in this protocol.

When a peer has established communication with another peer, it can stream bytes directly — this is most useful for applications like video or audio, or real-time "multiplayer" communication.

The published state, and the method that should publish a packet, are for indirect communication — it may be used to get set up for direct communication, or for a chat or social app for example. An application should publish `Packet<Publish>` when the intended recipient is known to be offline or does not respond to a ping. The packet will be encrypted and eventually, it will be received by the intended recipient who also subscribes to the `clusterId`.

##### 4. Topology

##### 4. Procedure

- LET `p` be an instance of `Packet<Publish>`
- IF `Peer A` is publishing a packet

- `Peer A` will construct a `Packet<Publish>`

- SET `p.clusterId` to its own cluster id
- IF a packet's message is larger than `1024 bytes` (In consideration of UDP's MTU)

- The packet must be split into smaller packets and each packet must be assigned `p.index`

- SET `p.packetId` to `sha256(p.previousId + p.message + p.nextId)` (`p.nextId` may be empty)
- SET `p.hops` to `0`

- `Peer A` must increment its own `.clock` property by one
- `Peer A` must select 3 peers to send the packet to

- IF `Peer A's NAT type is `Hard`

- exclude peers with NAT types that are `Hard`

- Sort peers by `.distance` (the last delta of time between a ping and pong)

- ELSE IF `Peer A` is receiving a published packet

- IF `Peer A` has already seen the packet with `p.packetId`, it should discard the packet

- this state ends here

- ELSE `Peer A` has never seen the packet it should add it to its cache

- `Peer A` should select 3 random peers to replicate the packet to
- `Peer A` must choose the larger `.clock` value (its own, or the value in the message)
- `Peer A` must increment its own `.clock` property by one

- IF `p.hops > 16` the state should end here
- IF `Peer A`'s `.clusterId` is equal to `p.clusterId` decrement `p.hops` by `2`
- ELSE decrement `p.hops` by `1`
- `Peer A` must select 3 peers to send the packet to

- IF `Peer A's NAT type is `Hard`

- exclude peers with NAT types that are `Hard`

- exclude the peer that sent `p`

#### STATE_5 (Join)

##### 5. Description

A peer sends the `Packet<Join>` packet when it comes online. This procedure will introduce peers and prompt them to go to `STATE_3` (Intro).

##### 5. Topology

##### 5. Procedure

- LET `Peer A` be the peer that wants to join the network
- LET `Peer B` be a peer that is outside `Peer A`s NAT
- LET `Peer C` be a peer that is known to `Peer B`
- LET `p` be the `Packet<Join>` sent by the remote peer
- LET `p.message.clusterId` be the cluster that the peer wants to join
- `Peer A` sends `p` to `Peer B`

- `Peer B` MAY concurrently enter into `STATE_6` if it has a NAT type of `Easy` or `Static`
- `Peer B` prepares `Packet<Intro>` to introduce the peer sending `p` and a selection of known peers

- IF a peer has a NAT type of `Hard` and `p.message.natType` is `Hard`, an introduction will not be made

- `Peer B` MAY concurrently send

- `Packet<Intro>` to `Peer A` that contains the info about `Peer C`
- `Packet<Intro>` to `Peer C` that contains the info about `Peer A`
- IF `Peer C` receives `Packet<Intro>` from `Peer B` it will enter into `STATE_3`
- IF `Peer A` receives `Packet<Intro>` from `Peer B` it will enter into `STATE_3`

#### STATE_6 (Exchange)

##### 6. Description

After two peers have established the ability to exchange packets. They may enter into an exchange state. This state is started when a peer sends a `Packet<Query>` and it is received by another peer.

##### 6. Topology

##### 6. Procedure

- LET `Peer A` be the peer sending the query
- LET `Peer B` be the peer receiving the query
- LET `p` by the `Packet<Query>` packet sent by the remote peer.
- IF `Peer B`'s cache includes `p.packetId`

- IF `p.message.tail` is `true`

- IF `Peer B`'s cache contains `p.packetId` and has the previous packet

- send `Peer A` the packet as `Packet<Answer>`, include the prior `.previoudId` as a suggestion

- ELSE send `Peer A` the next packet as `Packet<Answer>`, include the following `.packetId` as a suggestion

- IF `Peer A` receives a suggestion, it may decide to continue as many times as it wants
- `Peer A` MAY send 3 random packets to `Peer B` which has the lowest number of hops in the cache
- `Peer B` MAY send 3 random packets to `Peer A` which has the lowest number of hops in the cache

### Notes
>

This work derives from the paper published by Bryan Ford (baford@mit.edu), Pyda Srisuresh (srisuresh@yahoo.com), and Dan Kegel (dank@kegel.com) entitled "Peer-to-Peer Communication Across Network Address Translators".
