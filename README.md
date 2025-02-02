# Renet
[![Latest version](https://img.shields.io/crates/v/renet.svg)](https://crates.io/crates/renet)
[![Documentation](https://docs.rs/renet/badge.svg)](https://docs.rs/renet)
![MIT](https://img.shields.io/badge/license-MIT-blue.svg)
![Apache](https://img.shields.io/badge/license-Apache-blue.svg)

Renet is a network library for Server/Client games written in rust. Built on top of UDP,
it is focused on fast-paced games such as FPS, and competitive games that need authentication.
Provides the following features:

- Client/Server connection management
- Authentication and encryption, checkout [renetcode](https://github.com/lucaspoffo/renet/tree/master/renetcode)
- Multiple types of channels:
    - Reliable Ordered: garantee ordering and delivery of all messages
    - Unreliable Unordered: messages that don't require any garantee of delivery or ordering
    - Block Reliable: for bigger messages, such as level initialization
- Packet fragmention and reassembly

## Plugins

Checkout [bevy_renet](https://github.com/lucaspoffo/renet/tree/master/bevy_renet) if you want to use renet as a plugin with the [Bevy engine](https://bevyengine.org/).

## Visualizer

Checkout [renet_visualizer](https://github.com/lucaspoffo/renet/tree/master/renet_visualizer) for a egui plugin to plot metrics data from renet clients and servers:

https://user-images.githubusercontent.com/35241085/175834010-b1eafd77-7ea2-47dc-a915-a399099c7a99.mp4

## Echo Example
##### Server

```rust
let socket = UdpSocket::bind("127.0.0.0:5000").unwrap();
// Create one channel for each type reliable (0), unreliable(1), block(2)
let connection_config = ConnectionConfig::default();
let max_clients = 64;
let mut server: UdpServer = UdpServer::new(max_clients, connection_config, socket)?;
    
let frame_duration = Duration::from_millis(100);
loop {
  server.update(frame_duration)?;
  while let Some(event) = server.get_event() {
    match event {
      ServerEvent::ClientConnected(id) => println!("Client {} connected.", id),
      ServerEvent::ClientDisconnected(id) => println!("Client {} disconnected: {}", id)
    }
  }

  for client_id in server.clients_id().iter() {
    while let Some(message) = server.receive_message(client_id, 0) {
      let text = String::from_utf8(message)?;
      println!("Client {} sent text: {}", client_id, text);
      server.broadcast_message(0, text.as_bytes().to_vec());
    }
  }
        
  server.send_packets()?;
  thread::sleep(frame_duration);
}
```

##### Client
```rust
let socket = UdpSocket::bind("127.0.0.1:0")?;
// Create one channel for each type reliable (0), unreliable(1), block(2)
let connection_config = ConnectionConfig::default();
let server_addr = "127.0.0.1:5000".parse().unwrap();
let mut client = UdpClient::new(socket, server_addr, connection_config)?;
let stdin_channel = spawn_stdin_channel();

let frame_duration = Duration::from_millis(100);
loop {
  client.update(frame_duration)?;
  match stdin_channel.try_recv() {
    Ok(text) => client.send_message(0, text.as_bytes().to_vec())?,
    Err(TryRecvError::Empty) => {}
    Err(TryRecvError::Disconnected) => panic!("Channel disconnected"),
  }

  while let Some(text) = client.receive_message(0) {
    let text = String::from_utf8(text).unwrap();
    println!("Message from server: {}", text);
  }

  client.send_packets()?;
  thread::sleep(frame_duration);
}

fn spawn_stdin_channel() -> Receiver<String> {
  let (tx, rx) = mpsc::channel::<String>();
  thread::spawn(move || loop {
    let mut buffer = String::new();
    std::io::stdin().read_line(&mut buffer).unwrap();
    tx.send(buffer).unwrap();
  });
  rx
}

```
 
