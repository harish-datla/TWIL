# UDP vs TCP

A shooter sends the world's state 20 or more times a second, and a state that arrives late is already useless. Fast-paced games run that realtime stream over UDP, fire-and-forget packets with no delivery or ordering guarantee, not TCP, a reliable, in-order byte stream. TCP guarantees reliable, in-order delivery, and that guarantee is the problem: a single lost packet causes head-of-line blocking. TCP won't hand over any later data that already arrived until the retransmit lands (≥1 RTT later), freezing the stream in whichever direction lost the packet[2].
"Never use TCP" is wrong; "never for time-critical data" is right

TCP is fine for non-realtime channels: login, matchmaking, chat, patch downloads, REST calls. Slower genres (MMOs, turn-based games) ship whole gameplay streams on it. The precise rule (Fiedler's) is never use TCP for time-critical data. For the realtime stream, fast-paced games build a thin partial-reliability layer on UDP: sequence numbers plus an ack bitfield (each packet carries the latest received sequence and a 32-bit field of the prior 32, so every ack is effectively sent 32 times for redundancy), with unacked reliable messages re-included in outgoing packets until they are acked, so a lost packet never blocks the realtime stream[3].

Worked example

A client receives 60 state packets a second over a 100 ms round trip. Over TCP, one lost packet holds back everything behind it until the resend lands, at least 100 ms later: 6 packets' worth of newer state sits in the receive buffer, and the game freezes, then jumps. Over UDP the game uses packet 1,001 as soon as it arrives and never misses 1,000. For the acks, a packet carrying "latest received: 1,000" plus a 32-bit field for 999 down to 968 acknowledges up to 33 packets at once, so the sender learns about packet 1,000 from any of the next 32 packets that get through.
