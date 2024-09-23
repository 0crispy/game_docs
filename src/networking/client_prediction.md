# Client-side prediction

## The problem

So let's say you've implemented player movement.
Everything is working, the player is sending the position
to the server and the server is sending your position to others.
This is called client-authorative movement.
Everybody is happy, right? Well, actually there's a huge problem:
malicious users, meaning "What's stopping me from sending any position I want?".
This system basically allows cheaters to change the speed, teleport or even fly
(unless we do some sort of checking, which is very difficult),
because the server gets your position and just sends it to everyone.    
<p style="text-align: center;">
<img src="/img/client_authorative.png" width="60%">
</p>


![bruh](/img/client_authorative.png)


## How do we fix that?

Well, we make the server responsible for the positions of clients.
Instead of the clients sending their positions, the clients send
their input (keyboard, mouse) and then the server runs the same
movement scripts to change their position. After that, the server
send the positions to the clients. This is called server-authorative movement.
So it looks something like this:
<p style="text-align: center;">
<img src="/img/server_authorative.png" width="60%">
</p>

## Problem fixed?

Kind of. If you are happy with a input-to-movement delay, then yes.
But we want to go further. We want to eliminate the delay completely.
But first, let's make sure that we understand where the delay is coming from:

<p style="text-align: center;">
<img src="/img/movement_delay.png" width="60%">
</p>

So the delay consists of these parts:
1. The input packet has to travel from the client to the server
2. The server has to process the input (or wait for a fixed update)
3. The position packet has to travel from the server to the client

It seems like we created a new problem for ourselves.

##  The naive approach

The way we're going to solve this problem sounds quite simple:
we move the client with the current input **without** having to wait
for a position packet from the server. Unfortunately, the game world is
usually non-deterministic. Meaning that if we run two, virtually identical
simulations on the same world, in a sufficient amount of time the simulation
results will diverge. In simple words, the player positions of the server
and the client may be different, even with the same inputs.

That is why the client needs confirmation that their calculated position
is correct and identical to the one that the server calculated. This can be
partially fixed with a very naive approach (don't do that):
1. We receive a position from the server.
2. We compare our current position with the received one.
3. If the difference is too big, set the player's position to the received one. 
Slight problem:

> The position received from the server is **from the past**.  

Let me explain this by adding timestamps to the previous graph:

<p style="text-align: center;">
<img src="/img/naive_approach.png" width="60%">
</p>

As you can see, the player received a position with a 
timestamp of `1`, although the client's current time is `1.5`. 
And this graph shows exactly why that's a problem:

<p style="text-align: center;">
<img src="/img/position_is_past.png" width="60%">
</p>

1. Client sends the input packet `forward` and moves forward, so that `pos_y = 1`.
2. Right after that, the client another `forward` packet and moves forward, so that `pos_y = 2`.
3. The server receives the first input packet, moves the player and sends the position `pos_y = 1`.
4. The client receives a packet from the server, saying that `pos_y` should be `1`, although
our current `pos_y` is `2`.

That happened because the packet from the server was from the past, when the second input packet
was still on its way to the server.

## Server reconciliation

As it turns out, the naive approach doesn't work. This is what we have to do:
1. As a client, we store our positions in a list with a timestamp.
2. When we send an input, we also including the current timestamp.
3. When the server sends a position, it includes the timestamp that we sent.
4. When we receive a position and a timestamp from the server, we find the position with
the timestamp and compare that position with the one that the server sent.

Seems not too difficult. That's because the most difficult part is dealing with prediction errors.
If the positions don't match, we have to rewind time to the received timestamp and re-simulate
the whole world until we arrive at the current timestamp. And then, hopefully, we will have arrived
in a position that is mostly identical to the server's position.