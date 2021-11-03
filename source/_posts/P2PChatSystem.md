---
title: A Decentralized Chat Application
date: 2021-09-17 17:38:03
categories:
- Program
- Java
tags:
- Github
---

The project is to transform existing chat program into a decentralized chat application.

<!-- more -->

## Discussion About the Shout Feature

The shout command can be used by any user currently joined in a room. Once a peer shouted, all peers in the network includes all peers for which there is a path of connections from shouting peer are supposed to receive a provided message.

### Overall Design

Assume that we have established a peer network as shown in Figure 1. Peer C and peer D are connected to peer B. Peer B is connected to peer A. At a certain time, peer C types a shout command and then sends a Shout Command I to its upstream node peer B. After received the Shout Command I from its downstream node, peer B will send Shout Message to all the downstream nodes that connected to it. Also, peer B will check if it is connecting to other peers. If peer B does connect to another peer as shown in the figure, it will send a Shout Command II to its upstream node.  Then the peer A, as the upstream node of peer B, will do the same thing that is sending Shout Messages to its downstream nodes and Shout Command II to its upstream nodes. In this way, the broadcast of shout messages is achieved.

This is a brief overall design on Shout feature. The content and format of the command packet and message packet which involve in have been illustrated in the Figure 1.

![Figure 1](/img/P2PChatSystem/Picture1.png)

### Implementation Details

This part focus on the implementation details about the Shout feature. In this project, we are supposed to combine the server part and the client part into one peer. Therefore, I use two threads to handle the different parts separately.

![Figure 2](/img/P2PChatSystem/Picture2.png)

For the first step, when a peer wants to shout, it will send Shout Command I to its upstream peer through client handle thread. When upstream node’s server thread received a Shout Command, it will first check if there is an identity field in the command packet to distinguish the command’s type. If it’s a Shout Command I, it means that one of the peers connected to the peer wants to shout. Then for the second step, the peer will find the identity of the shout peer and send Shout Message containing the identity of the shout peer to all peers connected to the current peer and joining a room. At the same time, server handle thread will check if the client handle thread is connecting to another peer other than itself. If it does connect to another peer, then the client handle thread will send a Shout Message II containing the identity of shout peer to the upstream peer.

When a peer received a Shout Message from its upstream peer, there are two things to do. First, the client handle thread will print the shout message on the screen. Second, the peer will send the Shout Message to all the peers that connected to it and joined a room. When a peer received a Shout Command II, there are also two things to do. First, the peer will send Shout Messages filled with the information from the Shout Command II to all the peers which connected to the current peer and joined a room. Second, the peer will check if it is connected to another peer other than itself. If it is connecting to another peer, then it will retweet this Shout Message II to that peer. So on and so forth, the Shout Message can be broadcast through the peer-to-peer network.

### Discussion About the Shout Feature Implementation

In the previous description, I introduced the implementation of Shout feature. Overall, this implementation has its cons and pros when facing the general challenges of distributed system.

For scalability, the bottleneck is performance of the peer. Each peer needs at least two threads to handle the client and server side respectively. If the client uses the #connect command to connect to another peer, an additional thread is needed to receive message from the upstream peer’s server thread in real time. In addition, whenever a new peer connects to the current peer, the server thread creates a new thread to handle the keep-alive socket connect. So, this is where the paradox comes in. The use of threads allows us to easily scale the peer-to-peer network and at the same time becomes a bottleneck that limits the size of the network.

For concurrency, also threads help the system to control multiple processes with its unique competitive mechanism. The system meets the demand of high concurrency to a certain extent by combing multi-threaded technology with the multi-core and multi-threaded feature of CPU. This allows the system to perform several tasks simultaneously, improving operational efficiency and speeding up data processing. Data consistency is also ensured by lock or message queue techniques.
For failure handling, the implementation has many shortcomings. Only the simplest case is presented in the design and implementation, but the actual situation can be very complex. First of all, if a Shout Message or Shout Command is lost during transmission, then starting from the lost peer, subsequent peers will not receive the Shout Message or Command, and there is no means to detect the loss of the packet. Another problem is that if there is a cycle path among the peer-to-peer network, the broadcast of Shout Message will be no end.

![Figure 3](/img/P2PChatSystem/Picture3.png)

First consider a simple loop as shown in the Figure 3, which is a peer connected to itself. In such case, if the peer received a Shout Command I, it is supposed to send a Shout Command II to itself. This leads to an infinite loop. So, the system will check if the current peer is connected to itself, then decide whether it is necessary to send the Shout Command II packet.

When considering another slightly more complex loop as shown in the Figure 3-4, this broadcast loop issue will be difficult to solve. Peer A sends Shout Command I to peer C, peer C responses with Shout Message and sends the Shout Message to peer B. Also, peer C sends Shout Command II to peer D. In the meanwhile, peer D sends Shout Command II to peer A. After peer A receives the Shout Command II from peer D, it will transfer this Shout Command II to peer C. And so, the cycle continues endlessly.

![Figure 4](/img/P2PChatSystem/Picture4.png)

For security, there are several security issues in the implementation. First, all the packets are not encrypted, hackers could easily get some information through listening the communication channel. Second, hackers could hijack the packet, modify the content, and replay the new packet. Finally, hackers could create multiple peers and shout at a same time. The broadcast message may greatly increase the network load, and even crash the network.

## Discussion About the Decentralized Model

As discussed before, the peer will create two threads initially. The server thread is waiting for other peers’ socket connection and the client thread is used for handling user commands.

User could use #help command to get some help. The system will list all available commands. Only the peer owner could use #createroom command to create a new chat room on the current peer. User could use #list command to list all the chat room on the current peer. #who command is used for list all the members in a chat room. The peer owner could use #kick command to kick a peer from the current peer and the system will add its IP into blacklist. Therefore, the kicked peer is blocked from reconnecting. User could user #delete command to delete a room from current peer. #listneighbors command is used for listing the neighbors of current peer. #searchnetwork command will crawl over all the accessible peer in the peer-to-peer network automatically. User could use #quit command to quit the system.