---
title: "HuluParty: Remote Video Synchronizer"
date: 2020-10-21T15:34:30-04:00
categories:
  - blog
tags:
  - HuluParty
  - Chrome Extension
---

HuluParty is a Hulu video synchronizer created to allow multiple people to synchronize their shows/movies with eachother. It is made up of two components: a chrome extension and a web server. The chrome extension forms the interace and is responsible for creating sessions, and sending synchronization updates to the server. The web server is an intermediary that stores session data and delegates synchronization updates.

<div>
  <img style="text-align: left" src="/assets/gifs/shrek_gif.gif" width = "310" height = "200"/>
  <img style="test-align: right" src="/assets/gifs/shrek_gif.gif" width = "310" height = "200"/>
</div>

# In-Depth
> Before I continue I just want to note that this project is in no way affiliated with [Hulu Watch Party](https://help.hulu.com/s/article/watch-party).In fact, I finished this project almost two months before they released their offical version.

HuluParty was written in javascript and was built using a [Client-Server](https://en.wikipedia.org/wiki/Client%E2%80%93server_model) model. Both the chrome extension(client) and web server(server) communicate using [socket.io](https://socket.io/docs/v3/index.html),
a websocket library that allows for real-time, bidirectional event based communication.

In the following sections I will go over each component in detail.

## The Web Server
The server keeps track of all sessions. A session is what allows users to receive synchronization updates from eachother and can be created by any user. Sessions are stored in-memory and are indexed by unique IDs that the server is also responsible to creating. A typical session entry looks like the following.
```javascript
'84dba68dcea2952c': {
  sessionId: '84dba68dcea2952c', // uniqueID for session
  lastActivity: '2021-01-27T20:19:10.596Z', // the last time this session was updated
  lastVideoPos: 1800, // the last playback position of the video in milliseconds
  state: 'playing', // whether the video is playing or paused
  videoId: 123, // the ID Hulu assigns to all videos
  users: ['3d16d961f67e9792'] // a list of participating user IDs
}
```

Ther server also keeps track of all the users currently connected to the server. A user automatically connects to the server by activating the HuluParty extension while being on the Hulu website and can disconnect by leaving the website. Like sessions, users are also stores in-memory. A typical user entry looks like the following.
```javascript
  '3d16d961f67e9792': {
    userId: '3d16d961f67e9792', // uniqueID for the user
    sessionId: '84dba68dcea2952c', // the session this user is currently in. Initially this field is empty
    socket: socket // A reference to the websocket needed to communicate with this user
  }
```
The last main responsibility of the server is to respond to *events*. Events are the method of communication between client and server. The server responds to the following events which are sent by clients.

- `connect`: Sent when a new client connects. Prompts the server to create a new user.
- `createSession`: Sent when a client presses the create session button. Prompts the server to create a session.
- `joinSession`: Sent when a client wants to join a session. Prompts the server to add user to the session.
- `seekUpdate`: Sent when a client skips forward or backward in a video. Prompts the server to update the session info and send out an update to participating users.
- `playStateUpdate`: Sent when a client pauses or plays a video. Prompts the server to update the session info and send out an update to participating users.

## The Chrome Extension

The chrome extension forms the client part of this project and is responsible for reacting to user generated video updates like seeking the video or changing its play/pause state. Hulu doesn't have a public API, meaning that there is no official way to programatically communicate with or manipulate a video on its website. The client circumvents this by instead pulling information from and manipulating the [DOM](https://developer.mozilla.org/en-US/docs/Web/API/Document_Object_Model/Introduction) which is the HTML representation of a web page.

As mentioned before, the chrome extension acts as the interface for HuluParty. To create a session you first need be on a hulu video. The extension icon will then become "active", meaning it becomes clickable. After clicking on the extension icon the following popup will appear.

<img style="margin-left: 100px;" src="/assets/images/popup_cap.png" width = "410" height = "300"/>

Pressing the "create session" button will automatically place the invoking user in a session and also fill the textbox with a link. This link can be share with other Hulu users to take them to the correct video. After a user uses a join link to arrive at the correct video they can press the "join session" button which will then place them in the session. From then on, any video updates like seeking, playing or pausing will be sent and received by all users in the session. 

The code for the server can be found [here](https://github.com/kcharellano/huluparty-server) and the code for the client can be found [here](https://github.com/kcharellano/huluparty-client). If you made it this far, thanks for reading!
