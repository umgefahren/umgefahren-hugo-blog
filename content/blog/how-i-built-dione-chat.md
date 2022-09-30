---
title: "How i built the dione chat system"
date: 2022-03-23T17:32:24+02:00
draft: false
tags: ["tech", "web3", "dione"]
---
In the middle of the first wave of the pandemic a friend of mine wanted to build a basic P2P-messenger (like termchat) to improve his coding skills. About a year later I started writing my biggest project yet and I've been hooked since.

## I need a challenge project

In the summer of 2020, near the end of 11th grade, I worked with a friend for a start-up where we built a MVP for them. The product we delivered worked, but it was built horribly, built by teenagers
not knowing what they where doing (maybe I'll describe the tech stack once, but it is incredibly awful). After we finished the project, we reviewed on how we could improve. I decided to learn a new 
programming language, faster and better then Python (I went for Rust) and he decided to improve his general programming skills by building a big project in C.
I showed him [termchat](https://github.com/lemunozm/termchat), which we then went on using in school and we were fascinated by the idea. He continued to writing a similar project, which he never
finished, and I dug into the weeds on how termchat worked. In hindsight it is actually quite simple. It's actually just utilizing mDNS to find other users and then just connected to them. 
I liked the idea but wanted to build something people could use over the internet (basically IRC).

At this point it's important to point out how few I knew and still don't know about the topics I was dealing with back then and to this day. I'm a self taught programmer and i mainly did robotics and
Raspberry Pi programming with Python. I actually really learned programming to build a [Gaussian-Filter](https://en.wikipedia.org/wiki/Gaussian_filter) for a line following robot we made. 
So I knew some Linux, some Python, high-school mathematics and little to nothing about computer networks.

## Decentralization is incredibly hard

I soon figured I wanted to build something *like* [IRC](https://en.wikipedia.org/wiki/Internet_Relay_Chat) but without a central servers needed to communicate but also very anonymous. 
I know this sounds a lot like [Matrix](https://en.wikipedia.org/wiki/Matrix_(protocol)) but it's actually different.

I couldn't find a solution to the technical problems I was facing so I stopped working on it. I chased after the idea of a communication system that would split up messages in little packages and 
distribute the packages around the network in pseudo-random way. However I couldn't find a resilient way of distributing the packages, without passing the next location with the previous message 
(because that has obvious problems in consensus and reliability).

## Cryptography is key

When I looked into other messaging systems, I learned about the well-known [Signal Protocol](https://en.wikipedia.org/wiki/Signal_Protocol). The really interesting part about the Signal Protocol is
the double ratchet technique, that allows perfect forward secrecy. I was primarily interested in Double Ratchet because it still is the go-to way to implement end-to-end encryption 
(heck even WhatsApp uses it).
I later discovered, that a modified version of Double Ratchet would, in fact, solve a lot of my problems. But I will discuss that later.

## Kademlia is cool

I first learned about [Kademlia Distributed Hash Tables](https://en.wikipedia.org/wiki/Kademlia) in early December of 2020 and it took me actually some time until I realised how many problems it solved for me.
If you are unaware of Kademlia, I would encourage to take a look at it, because it is a computer science masterpiece but I will give a short breakdown here. Kademlia is a technique that powers most
modern P2P systems. It first appeared in the (in-)famous Bittorrent protocol and was key for it's success. While previous P2P filesharing services needed a central database of all file locations, 
Kademlia allows to address file **locations** through a distributed hash table that has, despite not being central, excellent properties.

It works like that:
Every node in a Dht (abbreviation for Distributed Hash Table) assigns an unique identifier. Typically this is just a bitstring, that has ideally some relation to the node. In Libp2p (and thus IPFS) 
it's a hash of the nodes public key. When the node joins the network it sends it's unique identifier. The peer node, in turn, performs a lookup in a local index of all nodes this node knows. It
returns the node and the node address, where the distance between the returned nodes and the asking nodes unique identifier is minimal. The distance function can be arbitrary here, but it's important,
that it's fast and well-defined. Typically it's just an XOR between the two. The asking nodes now contacts the returned node and repeats the process, until there is no better fit. It know places
itself at that location.

If a node want's to store something in the Dht, it generates a unique identifier for the piece of content it want's to store. The "saving node" know repeats the previous described process of finding
the closest node, to save the key-value-pair at the node with the least distance to the key.

The exact details of how routing is performed, discovery is handled and resiliency is implemented is somewhat complex in detail, but the principal rather elegantly simple. It allows to store,
retrieve and delete Key-Value pares in `O(log n)`, because it's essentially a binary tree.

This meant for my project that I just had to figure out some way of defining the keys for where the message packages would end up in a pseudo-random and deterministic manner without it being a problem
when messages where lost. However, as previously stated, it's not that simple to write a P2P network from scratch.

## My hate-love with Protocol Lab's libp2p

During research for a Rust implementation of Kademlia I encountered libp2p pretty early, but thought it was to complex and dismissed it. However it would turn out as my life saver. This was in January 2021.

What is libp2p? [Libp2p](https://libp2p.io) is a lower level framework for writing P2P applications. It was developed at and by [Protocol Labs](https://protocol.ai) for [IPFS](https://ipfs.io) but
was decoupled from IPFS and is now developed somewhat separately from IPFS. It defines a lot of protocols and behaviours necessary in P2P systems but also handles things like network IO.
There are several implementations of Libp2p. The most important one is the Go implementation, closely followed by the JavaScript and Rust implementations. The Go and JavaScript ones are primarily
driven by Protocol Labs themselves, where the Rust implementation is a slightly separate effort, primarily driven by the developers of [Substrate](https://substrate.io) a very powerful blockchain
framework responsible for Polkadot and funded by the Web3 Foundation.

Libp2p.rs meant for me, that I could solve a lot of complex problems quickly with Libp2p without having to worry about the weeds of it. However this is not that easy, because libp2p in general and 
the Rust implementation specifically suffer from a lot of problems.

Because the implementation is quite old, it doesn't embraces Async Rust that much (which is admittedly still very hard considering the problems with Async Traits) and therefore it has a very strange
and unintuitive *way* of doing things which takes a lot time to understand and makes it ugly to work with. Let me give an example.

Consider you want to use the Kademlia module of Libp2p and put values into your Dht and retrieve them afterwards.

First you have to define something called a Swarm Behaviour. This includes the protocols you want to use and a predefined Out Event, where all *events* get emitted to.
In the dione implementation this snippet looks like this:
```rust
#[derive(NetworkBehaviour)]
#[behaviour(event_process = false, out_event = "ComposedEvent")]
struct ComposedBehaviour {
	kademlia: Kademlia<MemoryStore>,
}
```
and the out event looks like this:
```rust
#[derive(Debug)]
enum ComposedEvent {
	Kademlia(KademliaEvent),
}
impl From<KademliaEvent> for ComposedEvent {
	fn from(event: KademliaEvent) -> Self {
		ComposedEvent::Kademlia(event)
	}
}
```
Not the trait implementing a conversion between a kademlia event and the composed event. If any of these parts are not perfect, it will just not work. The documentation is not helpful and the compiler
is not very useful. I only got it working with the help from the folks who wrote the thing.
This is actually the easy part. Now you need some way of retrieving your result. This is implemented with an Event Loop pattern and a lot of channels.
This is an extract of the struct defining the Event Loop:
```rust
pub struct EventLoop {
	swarm: Swarm<ComposedBehaviour>,
	command_receiver: mpsc::Receiver<Command>,
	pending_get_closest_peer: HashMap<QueryId, oneshot::Sender<anyhow::Result<PeerId>>>,
}
```
Notice the command receiver. At a different point in your code you have to define this channel and also a Command Enum that encapsulates everything you want to do. The sender part of this channels is
also your only way to use the protocols. You basically send a command, the command is received and the swarm is instructed to perform the lookup. You are given a QueryId you have to save, along with a
a channel to return the result. The command issuer is blocking on the receiving oneshot channel until there is some result.
This is an extract of the `run` part of the event loop:
```rust
pub async fn run(&mut self) {
	loop {
		tokio::select! {
			event = self.swarm.next() => {
				self.handle_event(event.unwrap()).await
			},
			command = self.command_receiver.recv() => match command {
				Some(c) => self.handle_command(c).await,
				None => return,
			}
		}
	}
}
```
and this is extract of the `event handle`:
```rust
async fn handle_event(
		&mut self,
		event: SwarmEvent<
			ComposedEvent,
			std::io::Error
		>
	) {
	match event {
		SwarmEvent::Behaviour(ComposedEvent::Kademlia(
		KademliaEvent::OutboundQueryCompleted {
			id,
			result: QueryResult::GetClosestPeers(Ok(GetClosestPeersOk { peers, key })), ..
		})) => {
			let key = libp2p::kad::kbucket::Key::from(key);
			let host_peer_id = *self.swarm.local_peer_id();
			let host_peer_key = libp2p::kad::kbucket::Key::from(host_peer_id);
			let host_distance = host_peer_key.distance(&key);
			let mut peer_id = peers.get(0).unwrap().to_owned();
			let remote_peer_key = libp2p::kad::kbucket::Key::from(peer_id);
			let remote_distance = remote_peer_key.distance(&key);
			if remote_distance > host_distance {
				peer_id = host_peer_id;
			}
			let _ = self
				.pending_get_closest_peer
				.remove(&id)
				.expect("Completed query to previously pending")
				.send(Ok(peer_id));
		}
	}
}
```
You can see this is a lot of very ugly Rust boilerplate you have to write to perform basic tasks. I will omit the `handle_command` function here, but you'll get the idea. This is not only very ugly
to write, it also peforms horribly, because commands to the swarm are issued very, very frequently and every time this results in a memory allocation for the oneshot channel. The complete code is 
also not very performant. I suspect that Libp2p is to blame here.

To make matters worse, Libp2p is not good documented. Some modules lack any documentation and the implementations vary a lot, so you basically are left to ask a lot of questions. Atypically for 
a Rust crate, the docs.rs of libp2p are also not very self explanatory, which is unusual for Rust crates.

But I don't want to rant here. It actually enabled me to build this hole project, so I'm very grateful.

## Solving a big problem

During the time I wrote my Abitur, actually three days prior to the Math exam, I implemented Double-Ratchet and X3DH with P-256 and published it on crates.io. It helped me understand everything
better and also how to write good Rust code.

After I graduated, when visiting my school for the last time, I had an epiphany: Double-Ratchet can be used to place messages in the Distributed Hashtable. If you use the output of Double Ratchet not
to encrypt a message, but instead use the bitstring as the key for a key-value pair, you can place packages in a peer-to-peer net with perfect forward secrecy and without breaking, when loosing messages
(however the Diffie-Hellmans are a problem but this can be solved). This idea was my breakthrough idea. I wrote the whole dione messaging system in my summer break, before going to university. 
It's about 7.000 lines of Rust code and you can actually visit it on [GitHub](https://github.com/Dione-Software/dione)! At time time of writing it is outdated, but it should still work!

I finished just a couple of days before I moved to Zurich, knowing that I actually succeeded. However the code should be considered a prototype. It barely works, it is not reliable and it's horribly
designed. I also chose a server-client pattern, which is actually not necessary but easier to implement. There was also a short time period where I deployed it on servers around the globe it worked for
the first couple of hours, but it crashed after that for yet unknown reasons. However these are just implementation issues and when I rewrite it, I will improve on that.

## Conclusion

This actually quite lengthy article concludes the journey of a self thought programmer high school student building a system to complex for the 2000s over the summer. I'm currently improving the
concept of dione, however I quit writing code for it. I still think that the solution is rather elegant and I will pursue the idea further.

Thank you for sticking so long with this text :)