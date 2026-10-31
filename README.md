# Ex.No:1a Study of Socket Programming

## Aim:

To perform a study on Socket Programming

## Introduction:

Socket programming is a crucial aspect of network communication, allowing for data exchange between computers over a network. It forms the backbone of various networked applications, enabling communication between clients and servers. This study explores the fundamental concepts of socket programming, its use cases, and provides a practical example to demonstrate its implementation.

## Understanding Socket Programming:

Socket programming involves the use of sockets, which serve as endpoints for communication. A socket is identified by an IP address and a port number, and it facilitates data transfer between a client and a server. The two main types of sockets are Stream Sockets, which provide a reliable, connection-oriented communication, and Datagram Sockets, which are connectionless and suitable for scenarios where reliability is less critical.

## Key Concepts in Socket Programming:

1.Sockets

- A socket is a software representation of a communication endpoint in a network.
- It is identified by an IP address and a port number.
- Sockets can be classified into two main types: Stream Sockets and Datagram Sockets.
- Stream Sockets provide a reliable, connection-oriented communication, while Datagram Sockets are connectionless and operate in a best-effort mode.

2. Client-Server Model

- Socket programming typically follows the client-server model.
- The server listens for incoming connections from clients, while clients initiate connections to the server.
- Servers are passive, waiting for connection requests, and clients are active, initiating communication.

3, TCP/IP Protocol:

- Transmission Control Protocol (TCP) and Internet Protocol (IP) are the foundational protocols for socket programming.
- TCP provides reliable, connection-oriented communication, ensuring data integrity and order.
- IP facilitates the routing of data between devices in a network.

4.Basic Socket Functions:

- Socket programming involves a set of functions provided by the operating system or programming language to create, bind, listen, accept, connect, send, and receive data through sockets.
- Examples of functions include socket(), bind(), listen(), accept(), connect(), send(), and recv().

## Server-Side Operations:

- Servers create a socket using socket() and bind it to a specific IP address and port using bind().
- They then listen for incoming connections with listen() and accept connections with accept().
- Once a connection is establi
- shed, servers can send and receive data using send() and recv().

## Client –Server Operations

Clients create a socket using socket() and connect to a server using connect().
After establishing a connection, clients can send and receive data using send() and recv().

## Use Cases of Socket Programming:

Socket programming finds applications in various domains, including web development, file transfer protocols, online gaming, and real-time communication. It is the foundation for protocols like HTTP, FTP, and SMTP, which power the internet. Socket programming enables the development of both server and client applications, facilitating the exchange of information between devices in a networked environment.

## Example Use Cases:

1. Web servers: Web servers use socket programming to handle incoming HTTP requests from clients, serving web pages and content.
2. Chat Application: Instant messaging and chat applications use sockets to enable real-time communication between users.
3. File Transfer Protocol: Protocols like FTP (File Transfer Protocol) utilize socket programming for transferring files between a client and a server.
4. Networked Games: Online multiplayer games rely on socket programming to facilitate communication between game clients and servers.
5. RPC mechanisms: which allow processes to execute code on a remote server, often use socket programming for communication.

## Program:

**Server:**

```python
import argparse
import socket
import threading
from datetime import datetime

clients = {}                 # username -> socket
lock = threading.Lock()      # protects the clients dictionary
LOG_FILE = "chat_log.txt"


def stamp():
    return datetime.now().strftime("%H:%M:%S")


def log(text):
    line = f"[{stamp()}] {text}"
    print(line)
    with open(LOG_FILE, "a", encoding="utf-8") as f:
        f.write(line + "\n")


def send_line(sock, text):
    sock.sendall((text + "\n").encode("utf-8"))


def broadcast(text, exclude=None):
    with lock:
        targets = [s for n, s in clients.items() if n != exclude]
    for s in targets:
        try:
            send_line(s, text)
        except OSError:
            pass


def handle_client(conn, addr):
    reader = conn.makefile("r", encoding="utf-8")
    name = None
    try:
        # ---- registration: ask for a unique username ----
        while True:
            send_line(conn, "Enter your username:")
            candidate = reader.readline().strip()
            if not candidate:
                return
            with lock:
                if candidate.isalnum() and candidate not in clients and candidate.lower() != "server":
                    clients[candidate] = conn
                    name = candidate
                    break
            send_line(conn, "Invalid or already taken (letters/digits only). Try again.")

        send_line(conn, f"Welcome {name}! Type /help for commands.")
        log(f"{name} joined from {addr[0]}:{addr[1]}")
        broadcast(f"[{stamp()}] * {name} joined the chat", exclude=name)

        # ---- main loop ----
        for line in reader:
            msg = line.strip()
            if not msg:
                continue

            if msg == "/quit":
                break
            elif msg == "/help":
                send_line(conn, "Commands: /list  /msg <user> <text>  /quit  /help")
            elif msg == "/list":
                with lock:
                    send_line(conn, "Online: " + ", ".join(sorted(clients)))
            elif msg.startswith("/msg "):
                parts = msg.split(" ", 2)
                if len(parts) < 3:
                    send_line(conn, "Usage: /msg <user> <text>")
                    continue
                target, text = parts[1], parts[2]
                with lock:
                    tsock = clients.get(target)
                if tsock is None:
                    send_line(conn, f"User '{target}' not found.")
                else:
                    send_line(tsock, f"[{stamp()}] (private) {name}: {text}")
                    send_line(conn, f"[{stamp()}] (private to {target}) {text}")
                    log(f"PRIVATE {name} -> {target}: {text}")
            elif msg.startswith("/"):
                send_line(conn, "Unknown command. Type /help.")
            else:
                log(f"{name}: {msg}")
                broadcast(f"[{stamp()}] {name}: {msg}", exclude=name)
                send_line(conn, "Acknowledgement Received")   # same ack as the basic version
    except (OSError, UnicodeDecodeError):
        pass
    finally:
        if name:
            with lock:
                clients.pop(name, None)
            log(f"{name} left")
            broadcast(f"[{stamp()}] * {name} left the chat")
        try:
            conn.close()
        except OSError:
            pass


def server_console(server_sock):
    """Lets the server operator type messages and commands."""
    while True:
        try:
            cmd = input()
        except EOFError:
            return
        if cmd == "/users":
            with lock:
                print("Online:", ", ".join(sorted(clients)) or "nobody")
        elif cmd.startswith("/kick "):
            target = cmd.split(" ", 1)[1]
            with lock:
                s = clients.get(target)
            if s:
                try:
                    send_line(s, "You were removed by the server.")
                    s.shutdown(socket.SHUT_RDWR)
                except OSError:
                    pass
            else:
                print("No such user.")
        elif cmd == "/quit":
            with lock:
                for s in clients.values():
                    try:
                        send_line(s, "Server is shutting down.")
                        s.shutdown(socket.SHUT_RDWR)
                    except OSError:
                        pass
            server_sock.close()
            return
        elif cmd:
            log(f"[SERVER]: {cmd}")
            broadcast(f"[{stamp()}] [SERVER]: {cmd}")


def main():
    p = argparse.ArgumentParser(description="Multi-client chat server")
    p.add_argument("--host", default="localhost")
    p.add_argument("--port", type=int, default=8000)
    args = p.parse_args()

    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)   # restart without "address in use"
    server.bind((args.host, args.port))
    server.listen(5)
    print(f"Server listening on {args.host}:{args.port}  (/users, /kick <name>, /quit)")

    threading.Thread(target=server_console, args=(server,), daemon=True).start()

    try:
        while True:
            conn, addr = server.accept()
            threading.Thread(target=handle_client, args=(conn, addr), daemon=True).start()
    except (KeyboardInterrupt, OSError):
        print("\nServer stopped.")
    finally:
        server.close()


if __name__ == "__main__":
    main()
```

**Client:**

```python
import argparse
import socket
import threading
import sys


def receive(sock, stop):
    reader = sock.makefile("r", encoding="utf-8")
    try:
        for line in reader:
            print(line.rstrip())
    except (OSError, UnicodeDecodeError):
        pass
    finally:
        print("Disconnected from server.")
        stop.set()


def main():
    p = argparse.ArgumentParser(description="Chat client")
    p.add_argument("--host", default="localhost")
    p.add_argument("--port", type=int, default=8000)
    args = p.parse_args()

    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    try:
        s.connect((args.host, args.port))
    except ConnectionRefusedError:
        print("Could not connect. Is the server running?")
        sys.exit(1)

    stop = threading.Event()
    threading.Thread(target=receive, args=(s, stop), daemon=True).start()

    try:
        while not stop.is_set():
            try:
                text = input()
            except EOFError:
                text = "/quit"
            if stop.is_set():
                break
            s.sendall((text + "\n").encode("utf-8"))
            if text == "/quit":
                break
    except (KeyboardInterrupt, OSError):
        pass
    finally:
        try:
            s.shutdown(socket.SHUT_RDWR)
        except OSError:
            pass
        s.close()


if __name__ == "__main__":
    main()
```

## OUTPUT:
### CLIENT:
<img width="955" height="360" alt="image" src="https://github.com/user-attachments/assets/ddcae173-156f-411f-bb51-89f74d7cf182" />



### SERVER:
<img width="957" height="380" alt="image" src="https://github.com/user-attachments/assets/7e351fb9-956b-4c35-ba44-d212f54b244a" />



## Result:

Thus the study of Socket Programming Completed Successfully
