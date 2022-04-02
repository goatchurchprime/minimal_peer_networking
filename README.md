# Godot Multiplayer networking workbench

This utility exposes the workings of the three highlevel multiplayer networking protocols (ENet, Websockets, and WebRTC) 
and has hooks to enable VR players to compress, transmit, unpack and interpolate their avatar movements across the network.

## Installation

https://docs.godotengine.org/en/stable/classes/class_networkedmultiplayerpeer.html#class-networkedmultiplayerpeer

download the webrtc libraries from here an put into webrtc directory:
https://github.com/godotengine/webrtc-native/releases

If having difficulties on linux, don't forget to try:
> sudo apt-get install libatomic1

If you are on Nixos, it needs patchelf to fix it:
> https://github.com/godotengine/webrtc-native/issues/44#issuecomment-922550575

## Operation

The **NetworkGateway** scene runs the entire process and is composed of a tree of UI Control nodes 
that can be used directly to visualize the state for debugging, or hidden behind another conventional multiplayer UI
such as a lobby.

The main script **NetworkGateway.gd** manages the choice of protocol and the connections, while **PlayerConnections.gd** manages 
the players spawning and removal.

### Network connecting

The toy example included is an ineffective pong game with the network provisioning code in the JoystickControls.gd script.
We connect using WebRTC at startup so it works out of the box.  This is done with the call to `NetworkGateway.initialstatemqttwebrtc()`

Signalling is all done through the the public broker connected to [HiveMQ](http://www.mqtt-dashboard.com/) and you can sniff 
out all the signals if you run the command:

> mosquitto_sub -h broker.mqttdashboard.com -t "tomato/#" -v

This dumps everything in the room `tomato` to the command line.  You can choose other rooms, so that connection 
can be like jit.si.  

The use of a public MQTT broker to initiate the connections means we can set the connection to "As necessary", which means 
that if there's live server on the channel it starts out as a server, otherwise it starts as a client and connects to it.
(Automatic handover code for when the server drops out is partly working, but unreliable, and could be finished if 
there is a sufficient use-case.)

You can select a different protocol (ENet or Websockets) when the Network is off, and then select server or client.
There will be UDP packets sent by the server to help any clients on the same router network to find and connect to it 
without needing to look up the local IP number.  (Or you can set this running on a external server with a fixed IP number 
on the internet)

### Players

By default it uses the path `/root/Main/Players` as the node that keeps the players together, and considers the first node in there 
as the **LocalPlayer**.

The LocalPlayer gets a **PlayerFrame** node the the **PlayerFrameLocal.gd** script associated to it.  
Any remote players that are created are included with the same, but with the **PlayerFrameRemote.gd** script attached to it.
These are the scripts which receive the player motions generated locally and unpack and animate the 
player motions remotely. 

These PlayerFrame nodes are what all the rpc() calls are made against.  The Player nodes are given consistent names 
across the network based on the networkID so that these rpc() calls, which depend on finding the same node in the tree across different 
instances in the game, are able to work.

The script attached to the Player (the node containing the PlayerFrame that visualizes the avatar) must have the following functions:

* func initavatarlocal(): Called at startup on the LocalPlayer

* func initavatarremote(avatardata): Called when a new RemotePlayer is created in the Players node

* func avatarinitdata() -> avatardata: The dict of data called on the LocalPlayer and sent to the function above

* func playername(): Used in the Networking UI to list the players

* func processlocalavatarposition(delta):  Called directly from the PlayerFrameLocal \_process() function before it reads the position

* func avatartoframedata() -> fd: dict of local player position state generated at each frame

* func framedatatoavatar(fd):  The unpacking of the remote player position state from the frame data.

* static func changethinnedframedatafordoppelganger(fd, doppelnetoffset): A function used to distort the set of frame data so it can be used as a player doppelganger 
to see how the motions would look on the other side of a network in real time.

To avoid a huge load on the network, the PlayerFrameLocal.gd and PlayerFrameRemote.gd scripts automatically thins down the 
data generated by avatartoframedata() and interpolates the gaps in the data for framedatatoavatar() respectively.
This depends on timestamps and estimates of network latency etc and is where the hard work needs to be done.

