# Mesh Design

## Applicability

Assumptions:
- Assumes spectrum is not being jammed
- Assumes only friendlies have access to the Channel key
- Assumes node IDs of friendly nodes are unique
- Assumes no-one is repeating entire LoRa packet as an attack?
- The encryption algorithm encrypts the entire message including header, except the rebroadcast flag. Any change to the data to be encrypted results in completely different encrypted data.
- Always sends ACKs for most messages, unless specifically requested in a minority of cases, as this is how paths are deemed reliable
- Do not send ACKs for ACKs, if behaviour like this is desired, send traceroutes instead

Limitations:
- Only those with the channel key (friendlies) can relay messages
- In the future a dynamic routing table would be much more flexible, this would allow specifying multiple recepients for the same message, rather than sending one multiple times. This is better than flooding, as once the message has been acknowledge by a recepient, they can stop trying to send specifically to that one. 

## LoRa Packet information

(_number transmission number_ may mean _unique packet ID_ is useless)

Each **message** packet contains the following information in a _header_:
- Packet type: **MESSAGE**
  - This is always present in all messages
- Message ID
  - Unique
  - Prevents replay attacks?
  - Enabled seeing if the same message is delivered via multiple paths
- Message number
  - Counts up from 0 for each message sent
  - (Could be changed to GPS time, but then this makes it relies on another system...)
  - Prevents replay attacks?
  - (Maybe _packet transmission number_ makes _message number_ redundant?)
- ~~Unique packet ID~~
  - ~~Used for tracking relays and retransmissions of relays~~
- Packet transmission number
  - Used for tracking relays and retransmissions of relays
- Message sender's node ID
- Packet sender's node ID (a.k.a. relayer, but this implies the first node is also a relayer, so the main term is better)
- Packet destination's node ID (a.k.a. next hop, _same comment as above_)
  - 0xFFFF for no particular path, or node name for specific node
  - (A single bit could be used to turn specification of packet destination off in the _packet type_ data (Packet type: **MESSAGE SPECIFIC PACKET DESINATION** vs Packet type: **MESSAGE FLOOD PACKET**), this would save a few bits when flooding, but waste a bit when not, but also make the node with organic id 0xFFFF not have to switch it's ID to share it with 0x0000)
- Destination's node ID
  - 0xFFFF for no particular node (flood to all)

## Examples

(_Knows they can receive packets sent by_ may be useless because a node can't influence who sends them packets, at least for now)

| Node | Can actually receive packets transmitted by | Knows they can receive packets transmitted by | Knows that packets they transmit are received by |
| - | - | - | - |
| Alice | Bob | | |
| Bob | Alice, Charlie | | |
| Charlie | Bob | | |

In this example we assume all nodes know of each other's existence and node IDs.

- Alice requests the message `Hello, world!` to be sent to to Charlie's node (equivalent of a 'Direct Message', even though anyone on the channel can see its contents, no need for public/private encryption as of now...). This is the first message Alice's node will send.
- Alice's node doesn't know a path to Charlie's node.
- Alice's node sends the following packet to get the message to Charlie:
  - Message ID: 0xC6BB
  - Sender's message number: _0_
  - Packet transmission number: _0_
  - Message sender: _Alice_
  - Packet sender: _Alice_
  - Packet destination: _No particular_
  - Message destination: _Charlie_
  - Message: `Hello, world!`
- Only Bob's node receives that packet. Bob's node doesn't know \ the best / a still valid / suitable \ path, and sends the following packet to relay the message:
  - Message ID: 0xC6BB
  - Sender's message number: _0_
  - Packet transmission number: _0_
  - Message sender: _Alice_
  - Packet sender: _Bob_
  - Packet destination: _No particular_
  - Message destination: _Charlie_
  - Message: `Hello, world!`
- Alice's and Charlie's nodes received this message. Alice knows the message has been relayed by Bob's node. Alice also sees that Bob's node doesn't specify a route to Charlie.
- Charlie's node decides (**need to specify how this decision is made**) that this message is enough, it shouldn't want to see if it receives the same message later, but with a stronger SNR. The variables used to choose the best path should ideally take the reliability and signal of all hops. At the moment it doesn't have enough information to make this kind of decision. Charlie's node sets the destination of the packet to be Bob, as it knows that it can hear Bob. It doesn't assumes that Bob can hear Charlie's node (**Whether to assume or not should be decided, is it safe to assume all node have the same setup and directionality? No**).
- This the the first ACK from Charlie's node
- Charlie's node sends the following packet: 
  - ACK ID: 0xE0A5
  - Sender's ACK number: _0_
  - Packet transmission number: _0_
  - ACK sender: _Charlie_
  - Packet sender: _Charlie_
  - Packet destination: _No particular_
  - ACK destination: _Alice_
  - ACK of: 0xC6BB
- Only Bob's node receives that packet.
