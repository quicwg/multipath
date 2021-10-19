# Multipath QUIC side meeting

Oct 18, 2021 - 15-17 UTC - online
--------

## Welcome (Mirja)

[All slides for today's meeting](https://github.com/mirjak/draft-lmbdhk-quic-multipath/tree/master/presentations)

* Not an official IETF meeting, but under Note Well, since it's preparing for an IETF contribution. 

* We have three active proposals, and we'd like to get to a point where we can consider WG adoption of a new document.

* The draft authors have been talking, and we're mostly on the same page with a few differences. We hope to have a -00 draft before the submission cutoff for IETF 112.

* Working on a new, merged draft, which can be found [on github](https://github.com/mirjak/draft-lmbdhk-quic-multipath).

## Scope and building blocks (Christian)

All the existing proposals have a common multipath core, plus scheduling, plus multipath extensions. 

The proposals do have very different ideas for some parts - address discovery, scheduling. 

We want to focus on the core multipath draft for now. 

There are things we want to do, that we don't need to do now. Examples - things like one-way paths.

Can we agree to work on the multipath core first?

There are building blocks in core QUIC now - connection migration, for example. Multipath needs to work with those building blocks. 

## Comparing different proposals (Quentin)

Focus is on core components.

Components of "multipath core"
* Negotiation
* Path establishment
* Mapping packet onto path,
* Closing path

What we agree on 
* Negotiation through QUIC transport parameters
* Multipath usage only for 1RTT packets
* Use path validation, just for multiple paths, and use multiple paths concurrently
* Focus on bidirectional paths

Closing paths is different between drafts - implicit signaling vs. explicit signaling. 

Numbering packets on paths - whether we use one number space for packet numbers, or one number space for each path, with per-path ACKs

Q&A 

* Hannu - we had a number of use cases - can we implement any of the known use cases with multipath core?
* Tommy - some use cases, yes - support core-first, to defer complexity to extensions
* Hannu - we can do multiple paths, that may be enough
* Spencer - there are use cases that have a lot of opinions about what packets go on what paths, but we can definitely do use cases like bandwith aggregation with core multipath

* Ian - core draft does everything I need to do for now. Multiple numbering spaces could work, but single space is less effort to implement. We need to figure out whether we need it (more discussion later)
* Apostolis - we have 3GPP requirement for QoS flows. Is that relevant here? 
* Mirja - all the QUIC packets on a connection have the same QoS but different paths could potential get different QoS 
* Spencer - 3GPP will probably end up with one QUIC connection per QoS, but that's a 3GPP conversation

## Packet Number Space(s) (Christian)

With multiple number spaces, we need an ACK frame that includes information about the path the packet was received on.

Both proposals that used multiple number spaces based the number spaces on AEAD nonce (IV + sequence number). Each packet is encrypted using AEAD.

Using the same packet numbers means you're using the same encryption key and initialization twice, so you are using the same key twice, and making your crypto easier to break. 

Christian has implemented one sequence number space and multiple sequence number spaces in picoquic.

Implementations that use null connection IDs in packets can't use multiple number spaces for those packets.

Single number space, if you're sending packets on different paths that have the same latency, you're fine, but different latencies can mean your packets arrive significantly out of order, and you have large numbers of packet ranges in your ACKs. 

High speed paths could give you thousands of ranges. Tested two paths with significantly different RTTs with a single number space, and you do get big packet ranges. Receiver would have to use some intelligence to compose ACKs. 

Some senders send packets in batches today, and that would give you smaller number of ranges. 

Christian also experimented with limited number of ranges per ACK, and limited number of times you send a range in an ACK. 

Loss recovery works with RACK if you have multiple number spaces. 

Nice slide on pros and cons between single space and multiple spaces. The point is that there are advantages to each approach.

Many servers require CIDs, so you know which server in the farm the packet goes to, but many clients don't use CIDs, because they can distinguish between endpoints based on four-tuples and reduce the size of each packet. 

Current new draft proposal allows for negotiation when the connection is established. Is that a way foward for now?

* Robin - why not use different keys per path? 
* Christian - Chicken and egg problem: You need keys for probes.

* Martin - I'm pushing QUIC offload. Changing the way crypto works will make it harder to deploy multipath.
* Mirja - It depends on how you do crypto offload, doesn't it? Nonce length need to be flexible. (Christian agrees - you just need a different IV for each connection)
 
* Ian - definitely prefer single number space, and suggest having the sender be smarter about ranges it sends - if you assign half the ACK space to each path, you don't have to change anything to avoid large numbers of ranges. 
* Christian - this is in the spirit of what I wrote in the single number space draft, but getting smarter is going to require experience. We want to have a negotiation for this. 

* Martin - is this per-direction? 
* Christian - yes, but then you need two codes, with complexity of both.

* Mirja - don't you need additional signaling to tell the other end what the division of packet number space is? And doesn't that get more complex with more paths?
* Ian - I'm suggesting that we keep the single ACK frame. I can allocate 4000 numbers at once, without the receiver knowing that. And I should have read Christian's draft :-) 
* Christian - Ian's goal is to have the smarts in the sender, and that's a nice goal, but we need experience to know that works.

* Yunfei - what if you have a large number of paths/numbering spaces?
* Christian - there are reasons to send a large number of packets at once. That's not hard to implement - it's in many stacks today. 

* Christian - expecting Robin to ask what multiple number spaces does to QLOG? 
* Robin - yes, but don't want QLOG to drive this decision.

* Maxim - SRP - we have multiple strategies in SRP (broadcast, active/standby, load balancing)
* Mirja - load balancing is definitely in scope. 

* Mirja - want to know what to put in the -00 draft for number spaces. 
* Yunfei - we used multiple number spaces in our experimentation, didn't need to change any recovery strategies. If you can use CID to identify a path, that makes things a lot simpler. 
* Mirja - sounds like the number of code changes for single number space is smaller, but the complexity required for some senders is greater. 
* Mirja - looks like we can put both options in -00 of the working group draft and have the goal of deciding on one option before publication.

* Quintin - if we have a lot of paths, what will that look like?

* Christian - I think the sense of the room is that we'd like to pick one alternative for numbering spaces, but we're not there yet. Do we have to keep null CIDs? How much intelligence is required? 

* Christian - if you have 64 paths, you probably have some paths that aren't getting much use, but we just don't have enough experience to know which one to choose yet. And null CID is the only blocking point. 

## Next steps and way forward in the QUIC wg (Mirja)

**We'll work on having one -00 document for core multipath that we can ask the QUIC working group to adopt.**

That's what "way forward" means for QUIC multipath, for now. 

Please comment and provide feedback on GitHub for the -00 version!

## Experimental results (Yanmei&Yunfei)

Motivations - verify potential performance gain and understand the challenges of running multipath. Implemented in separate packet number spaces with QoE control in multi-path scheduling.

Q&A 

* Mirja - what was your experience with multiple number spaces?
* Yunfei - that was easier for us (because we don't need to change recovery and rtt measurement logic. Tunning performance can be time consuming). Second benefit was more explicit path management. You can manage the path status of a path or close a path by sending path_status from on another path, which makes further extensions of multi-path features more feasible.  
* Lucas - thank you for bringing this work to the QUIC working group at IETF 111 (as "if time permits" there), and thank you for presenting it here. It's good that people are seeing it now. 

## Finishing up

Lucas - thank everyone for bringing this work forward, and getting closer to one proposal.

Lucas - Do the other components we talked about need to come to the working group at the same time?

Mirja - We want to focus on the core multipath document for now

Lucas - Can someone bring a summary report for this meeting to the QUIC working group at IETF 112?

Lucas - this is encouraging (no promises, of course)

Zahed - agree that this is encouraging. This isn't an easy problem to solve. 

Spencer = is there anything that the QUIC working group needs to do, in order to be able to adopt work in this space? 

Lucas - we have a more general QUIC working group charter, now that QUICv1 has been published. No charter update should be required for the scope of multipath that's been discussed today. (Multipath congestion control schemes, for instance, would be a different story!)

Participants (extract based on voluntary sign-up; total count was 40):
* Mirja Kühlewind
* Martin duke
* Spencer Dawkins
* Robin Marx
* Tommy Pauly
* Lucas Pardue
* Olivier Bonaventure
* Nicolas Kuhn
* Adi Masputra
* Florin Baboescu
* Magnus Westerlund
* Gorry Fairhurst
* Quentin De Coninck
* Emile Stephan
* Michael Eriksson
* Behcet Sarikaya
* Apostolis Salkintzis
* Chi-Jiun Su
* Maxim Sharabayko

## Chat from Teams during the call

[17:06] Martin Duke
Are these slides posted anywhere

[17:08] Mirja Kuehlewind
https://notes.ietf.org/multipath-quic-side-meeting-Oct-2021 

[17:08] 
Qing An (Guest) no longer has access to the chat. 

[17:10] Yunfei Ma (Guest)
https://github.com/mirjak/draft-lmbdhk-quic-multipath/tree/master/presentations 

[17:17] Martin Duke
spencer and behcet, please mute

[17:17] Magnus Westerlund
I muted behcet. 

[17:17] Tencent - Spencer Dawkins (Guest)
thanks, Martin.

[17:20] Flinck, Hannu (Nokia - FI/Espoo)
RH

[17:25] Flinck, Hannu (Nokia - FI/Espoo)
+q

[17:26] Tommy Pauly (Guest)
+q

[17:27] Tencent - Spencer Dawkins (Guest)
+q 

[17:29] Ian (Guest)
+q

[17:30] Apostolis Salkintzis
+q

[17:31] Christian Huitema (Guest)
we will have the discussion of multiple PN in a minute 

[17:33] Tencent - Spencer Dawkins (Guest)
+q

[17:34] Behcet (Guest)
Mirja, please send notes page URL

[17:34] Mirja Kuehlewind
https://notes.ietf.org/multipath-quic-side-meeting-Oct-2021

[17:35] Robin Marx
https://notes.ietf.org/multipath-quic-side-meeting-Oct-2021?edit

[17:35] 
behcet (Guest) no longer has access to the chat. 

[17:45] Robin Marx
Just out of interest: what's the reason we can't use a per-path label for deriving a new TLS payload key?  

[17:45] Martin Duke
+q

[17:47] Nick Banks
Robin, I assume that you would have to encode that somehow in the packet then. Otherwise, how would you know which identifier to decrypt the packet?

[17:47] Martin Duke
the CID

[17:48] Robin Marx
Nick Banks that's the same problem as determining which nonce to use, no? you use the CID, like Martin says

[17:49] Nick Banks
But if you're dependent on the CID, then you're back to the problem where it all breaks down when a NULL CID is used.

[17:49] Martin Duke
With multipath you have to use a nonzero CID, no?

[17:49] Robin Marx
Didn't Christian mention that's a problem with the nonce just now? Or did I misinterpret that? 

[17:50] Mirja Kuehlewind
you have to use the CID for the nonce , not just the packet number as in single path quic

[17:50] Robin Marx
I don't mean as solution to the NULL CID problem, I mean as a solution to need the larger nonce

[17:50] Nick Banks
No, a CID is only required if the UDP tuple alone isn't enough to identify the connection

[17:50] Nick Banks
MsQuic for instance, only uses a CID when more than one connection is sharing a socket.

[17:50] Ian (Guest)
+q

[17:50] Nick Banks
(which is not the default behavior)

[17:51] Martin Duke
but by definition a different path is not identifiable by 4tuple

[17:51] Nick Banks
Identifiable by who?

[17:51] Martin Duke
the receiver

[17:51] Nick Banks
The connection can have N sockets that it does not share.

[17:51] Nick Banks
Each socket is a one to one mapping to a connection

[17:51] Nick Banks
So you still don't need a CID

[17:53] Robin Marx
+q

[17:53] Martin Duke
so a packet arrives with a new source address and no CID; how do I know what keys to use?

[17:53] Mirja Kuehlewind
you need to have a CID

[17:54] Ian (Guest)
My comment/question was that it's easier to be smart on the sender side than receiver.

[17:54] Ian (Guest)
ie: send in PN batches of 256 would be an easy approach.

[17:54] Nick Banks
You control the CID used for packets you receive. If you need a CID, then you give it to your peer.

[17:54] Ian (Guest)
If you knew you only had 2 paths, you could just allocate alternating blocks.

[17:54] Nick Banks
But if you have a new socket for each path you're using, then you don't need the CID.

[17:54] Martin Duke
q (clarifying question)

[17:55] Ian (Guest)
RFC9000

[17:56] Behcet (Guest)
+1 to Ian

[17:59] Ian (Guest)
Originally, I was thinking you'd allocate the first 32 bits(ie: 4 billion) packets for one path, the second for the next.  But that makes the PN larger.  So there is a trade-off.

[17:59] Ian (Guest)
But it completely fixes the immediate ack for out of order problem. 

[17:59] Mirja Kuehlewind
if you split the PN space it's basically like multiple PN spaces

[18:00] Ian (Guest)
But way easier to implement.

[18:00] Ian (Guest)
+q

[18:00] Nick Banks
Well, it's a little different because there aren't overlapping numbers.

[18:00] Nick Banks
Which is what broke packet encryption

[18:00] Nick Banks
And then required the extra nonce

[18:01] Nick Banks
Great point Martin!

[18:01] Ian (Guest)
+1 to Martin 

[18:04] Martin Duke
+q

[18:06] Mirja Kuehlewind
+q

[18:07] Rui Paulo (Guest)
If you allocate blocks of 256 PNs for each path and you send on both paths simultaneously, the receiver will send ACKs immediately much more often than once a in while Ian :-)

[18:07] Ian (Guest)
Not if the receiver tracks largest received per path, which is my other suggestion. 

[18:07] Ian (Guest)
It'd be every 256 packets

[18:08] Ian (Guest)
Which seems okish

[18:08] Ian (Guest)
But you can pick a larger number if you're using an ACK frequency of >256 

[18:08] Rui Paulo (Guest)
yeah that's a small receiver change that is doable 

[18:08] Yunfei Ma (Guest)
+q

[18:12] Michael Eriksson
I think it's easy to wait for 64k to be available in the cwnd and then send a full GSO batch -> lots of packets in sequence

[18:12] Ian (Guest)
Add an int for the largest allocated, and then allocate in blocks.

[18:13] Ian (Guest)
And two ints for the start and end of the current range.

[18:13] Maxim Sharabayko
+q
[18:13] Ian (Guest)
per path.

[18:16] Behcet (Guest)
single packet space

[18:16] Nick Banks
+1 single packet space

[18:17] Fairhurst, Dr Godred (Engineering)
+q
[18:17] Robin Marx
multiple spaces is definitely easier for qlog/qvis ;) 

[18:18] Olivier Bonaventure
I'm still in favor of multiple packet number spaces

[18:18] Yunfei Ma (Guest)
+q
[18:20] Ian (Guest)
Supporting both risks having most implementations only support one or the other.
like 1

[18:20] Robin Marx
+1 Ian
[18:20] Rui Paulo (Guest)
+2 Ian

[18:21] Ian (Guest)
We can punt this decision down the road, but we need to end up on 1 eventually. 

[18:22] David Schinazi (Guest)
+1, standardizing both is a recipe for non-interoperability. We don't need to pick today, but we need to pick a single one before publication

[18:23] Yanmei Liu (来宾)
+1 David 

[18:24] Lucas Pardue (Guest)
we don't need to decide now, but we should state the goal to pick only one before publication 

[18:24] Lucas Pardue (Guest)
and if there is compelling reason to revisit that goal, we can

[18:24] Nick Banks
IMO, Martin's comments about HW offload are really important. Changing anything to how packet encryption/decryption will make HW offloads harder.

[18:25] Michael Eriksson
Nick: +1

[18:25] Nick Banks
Especially since this option is negotiated after the handshake. Then you're in a world where the HW would have to support both modes (some connections have it negotiated and some don't).

[18:25] Nick Banks
That'd be a huge pain.

[18:25] Nick Banks
Also, MsQuic would use NULL CIDs if it's supported.

[18:26] Ian (Guest)
We would too

[18:26] Rui Paulo (Guest)
Same here.  We use NULL CIDs today already

[18:27] Tencent - Spencer Dawkins (Guest)
+q 

[18:29] Nick Banks
Just to confirm, https://github.com/mirjak/draft-lmbdhk-quic-multipath is the GitHub for this work now? 

[18:30] Quentin De Coninck
yes

[18:31] Robin Marx
Christian Huitema (Guest) (gast) I'm not sure I understand the chicken/egg problem for using a per-path label derivation. IIUC the larger nonce just uses the CID seq nr, which you could also use as a basis for the key derivation? So I'm not sure why the label-approach would have chicken and egg, but the larger-nonce wouldn't? (Not that I don't believe you, just trying to understand ;-) )

[18:32] Tencent - Spencer Dawkins (Guest)
Could someone else take notes on Yanmei's presentation for a few minutes? I need to do some cleanup while I remember what I didn't capture yet.

[18:32] Mirja Kuehlewind
Thanks everybody for the discussion! As I said please review the new draft in github and also provide more comments there!

[18:33] Mirja Kuehlewind
@spencer not sure if you need to minute this part of the presentation; I guess the slides capture most of it

[18:35] Tencent - Spencer Dawkins (Guest)
I'll be back for Q&A 

[18:38] Rui Paulo (Guest)
Robin Marx Wouldn't that require trial decryption? 

[18:39] Christian Huitema (Guest)
That's great work. We need more studies of when multipath is actually worse than single path, and what to do about it, e.g., reinjection.

[18:39] Rui Paulo (Guest)
Perhaps not. I guess you're suggesting that the receiver looks at the CID on the packet and then finds the seq number.

[18:40] Olivier Bonaventure
> Christian Huitema (Guest)
> That's great work. We need more studies of when multipath is actually worse than single path, and what to do about it, e.g., reinjection.
There are many MPTCP papers that are relevant, but you need to consider a range of network conditions 

[18:40] Robin Marx
Rui Paulo (Guest) (gast)indeed... lookup the seq nr and then lookup the correct key for the per-seq-nr label 

[18:41] Lucas Pardue (Guest)
+q

[18:43] Tencent - Spencer Dawkins (Guest)
+q

[18:43] Robin Marx
Christian Huitema (Guest) (gast) mentioned you need the keys for sending probes, but I'm not getting how that's not true for the large-nonce approach as well 

[18:45] Yanmei Liu (来宾)
Thanks, Lucas :)

[18:46] Christian Huitema (Guest)
Large nonce is mechanical. CID-sequence number is fund from the CID in the incoming packet.

[18:47] Robin Marx
but if you use the CID seq nr as part of the key derivation, that's the same concept no?  

[18:48] Christian Huitema (Guest)
More costly. Setting up a key is definitely more costly than changing a nonce. 

[18:49] Lucas Pardue (Guest)
thank you all

[18:49] Robin Marx
<thumbs up>

[18:49] Zaheduzzaman Sarker
+1 to Lucas on chater..

[18:50] Apostolis Salkintzis
Thanks Mirja and everybody. Very useful discussion!

[18:50] Yanmei Liu (来宾)
Goodbye~ 


