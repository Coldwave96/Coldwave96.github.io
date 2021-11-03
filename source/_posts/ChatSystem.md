---
title: A Simple Chat System
date: 2021-09-17 17:38:03
categories:
- Program
- Java
tags:
- Github
---

This is a simple example of distributed system implementation.

<!-- more -->

## 1.Introduction

The project is to create a C/S architectural model-based chat system. The chat system consists of a chat server and one or more chat clients. The chat server is able to accept multiple incoming TCP connections. The chat server can create a new chat room and move between existing chat rooms. Messages sent by a chat client will be broadcast to all clients which are in the same chat room.

## 2.Protocol Implementation

There are 8 protocols needs to be implemented in this chat system. At the beginning when a client connects to the server, the server will response with some initialized messages. After the socket is established, the client would be able to change identity, join rooms, ask for room list and room content information, create room, delete room, send messages, and quit the service. The following figures show how the server and client react to the eight protocols.

![Protocol diagram of the chat server](/img/ChatSystem/Picture1.png)

![Protocol diagram of the chat client](/img/ChatSystem/Picture2.png)

### 2.1.Initialization

The server will listen to the specified port. Once a client connects to the server, the server will start a thread to handle this socket and do following things:

* 1.Responds with NewIdentity message.
* 2.Moves the client into MainHall.
* 3.Sends RoomChange message to all the clients in the MainHall.
* 4.Sends RoomList message to the client.
* 5.Sends RoomContent message to the client.

Illustrated by the following figure:

![Initialization between the server and clients](/img/ChatSystem/Picture3.png)

### 2.2.Identity Change Protocol

When client sends a IdentityChange message to the server, the server will respond with a NewIdentity message. If the client wants to change to an invalid or existed identity, the value of former and identity field in NewIdentity message is same. And this NewIdentity message is only sent to the corresponding client. Otherwise, server will broadcast this NewIdentity message to all clients in the same chat room.

![Identity Change Protocol between the server and clients](/img/ChatSystem/Picture4.png)

### 2.3.Join Room Protocol

When client sends a Join message to the server, the server will respond with a RoomChange message. If the client wants to join an invalid or non-existed room, the value of former and roomid in RoomChange message is same. And this RoomChange message is only sent to the corresponding client. Otherwise, server will broadcast this RoomChange message to all clients in the former and changed room. If client joins the MainHall, server will also send RoomList message and RoomComtents message after RoomChange message.

![Join Room Protocol between the server and clients](/img/ChatSystem/Picture5.png)

### 2.4.Create Room Protocol

When client sends a CreateRoom message, the server will respond with a RoomList message. If the client wants to create a valid room, the server will respond with a RoomList message with the new room in the list.

![Create Room Protocol between the server and clients](/img/ChatSystem/Picture6.png)

### 2.5.Room Content Protocol

When client sends a Who message, the server will respond with a RoomContents Message.

![Room Content Protocol between the server and clients](/img/ChatSystem/Picture7.png)

### 2.6.Room List Protocol

When client sends a List message, the server will respond with a RoomList message.

![Room List Protocol between the server and clients](/img/ChatSystem/Picture8.png)

### 2.7.Message Protocol

When client input anything except for the commands, the client will take the input as messages and send the Message to the server. Then server will broadcast the Message to all clients in the same room.

![Message Protocol between the server and clients](/img/ChatSystem/Picture9.png)

### 2.8.Delete Room Protocol

When client sends Delete message, the server will do the following things:

* 1.Check whether the room is existed, and the owner is the client or not.
* 2.If all conditions are met, server will move all users in that room to the MainHall and delete the room. Also, the server will send RoomChange message as well as RoomList message and RoomContents message to these clients.
* 3.Send RoomList message to the client which want to delete a room.

![Delete Room Protocol between the server and client](/img/ChatSystem/Picture10.png)

### 2.9.Quit Protocol

Client just sends a Quit message to the server then shut down the program. If the client owns a room, then the server will remove the owner of this room. After that server will remove this client from its current room and check if it is the last client in that room. Server will delete that room if that room is empty, and its owner is also disconnected from the server. At last server removes the client socket from the socket list.

## 3.Discussion About Concurrency

Concurrency is an important feature of distributed system. Both services and applications provide resources that can be shared by clients in a distributed system. In this chat system, some concurrency has been realized, but there are also some problems have not been resolved when dealing with some concurrency.

First let’s discuss about the concurrency that the system has implemented. When the server starts, it will listen on the specified port. Once there is a socket connection request from the client, the server will create a new thread to process it. At the same time, the main program of the server is still listening to the specified port and waiting for a new socket connection request from another client. Different socket connections are processed by different threads and do not interfere with each other, so services and applications allow multiple client requests to be processed concurrently.

However, the concurrency implemented in this system is extremely limited. Next let’s discuss the limitation of the current system. First, the system does not consider high concurrency. After the client establishes a socket connection with the server, the thread handling the socket connection will run until the client actively exits the system. Although threads may occupy very few resources, the performance of the server is limited, and thread resources are limited as well. This will make it difficult for the system to deal with high concurrency. Second, there is no lock mechanism for access to shared resources in the system. Therefore, theoretically, different processes may operate on the same resource at the same time, which may lead to conflicts and inconsistent results. For example, if two users want to create a room with the same name at the same time, unpredictable error results may occur. Therefore, lock mechanism should be introduced to ensure the safe use of shared resources in a concurrent environment.

Similarly, the client also processes its socket connection with the server by thread. The client needs to listen to the user's input and process the messages sent by the server at any time. This is the concurrency problem faced by the client. However, this chat system does not solve this problem, but processes user input and server pushed messages in the same thread. It may take two threads to deal with these two things separately. Then, like the server, this processing method also faces the same problem of accessing shared resource. In addition, if the user is entering something and the server pushes a message, it is also a problem how the separate thread scheme handles the situation.

## 4.Multi-server Architecture Design

For a multi-server chat system, we have to do the following:

* 1.High availability: No single node failure should cause service unavailability.
* 2.Easy to scale: Horizontally scalable, with the ability to adapt to different amounts of online users.
* 3.High concurrency and low latency: Be able to support a large number of users sending and receiving messages at the same time, with a delay of milliseconds from the message being sent to the delivery of all online ends.
* 4.Client compatibility: New applications are able to interoperate across multiple devices at the same time, such as web, mobile and desktop, and even smart TV.
Thus, the overall framework design is shown in the figure below.

![The overall framework design of multi-server chat system](/img/ChatSystem/Picture11.png)

The client layer is supposed to deal with compatibility issues with various devices, message channel management and maintenance, and data security.

The diversity of client implementation technologies leads to differences in the underlying data communication protocols between the client and the gateway. Therefore, the gateway layer is responsible for managing client connections, protocol conversion, logic for data security and efficient distribution of broadcast messages.

In addition to serving as a relay point for messages, the routing layer also assumes the role of load balancing and high availability. It is easier to expand capacity when the processing capacity of a single business node reaches a bottleneck. When a network failure occurs in a server cluster, it can be switched to the backup server cluster to ensure service availability.

The server layer handles the business messages of the chat system.
Now let’s deep dive into the multi-server chat system and focus on the message protocols.

For the client, it will try to connect to the gateway depending on the distance of the gateway in the list. Once the TCP keepalive connection established, the server can do the same thing as the current chat client.

There are two options for handling the communication between servers at router layer.

First option is to provide a master server. The master server has a router table. All messages are pushed to the master server, which distributes the messages to different service servers according to the router table. This places a high demand on the performance of the master server.

Another option is to implement a message queue based on a publish-subscribe system. The basic structure is shown in the figure bellow.

![Cross-server communication via message queues](/img/ChatSystem/Picture12.png)

Assume there is a client 1 login in at the ChatServer 1 and a client 2 login in at the ChatServer 2. ChatServer 1 and ChatServer 2 will subscribe to messages about “client 1” and “client 2”in the message queue respectively. If client 2 sends a message to client 1, the ChatServer 2 will publish the message with a flag “client 1” to the message queue. Then message queue will notify ChatServer 1 that someone sends a message to client 1. Chatserver 1 will access this message and send it to client 1.

Message queues also have disadvantages. First, once the message queue crashed, so doed the entire system. Second, message queue makes the system more complex. How to ensure that messages are not consumed repeatedly? How to deal with the case of message loss? How to ensure the sequential nature of message delivery? These are the challenges that need to be addressed. Finally, there is the issue of consistency. If a request requires multiple operations, but one of them fails. Although the feedback to the user is success, the request is actually not fully executed. In general, however, the advantages of message queues outweigh the disadvantages, so it is a better choice than the first option.
