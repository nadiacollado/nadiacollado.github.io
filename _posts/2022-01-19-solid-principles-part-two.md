---
layout: post
title:  "Writing an Echo Server that follows the SOLID principles"
date:   2022-01-19 14:57:41 -0400
categories: blog
---

**What is an Echo Server?**

An Echo Server is an application that allows a client and a server to establish a connection from which the server can “echo” back messages it receives from the client. The client and server communicate via sockets. Both parties bind sockets to their end of the connection, reading from and writing to the bound socket. 

Writing an Echo Server is a great way to put the SOLID principles to practice. 

**SOLID Principles**

The SOLID principles are the five basic principles of object oriented programming. Their aim is to make designs easier to understand, maintain, and extend. They do this by reducing dependencies within the code. 

The Single Responsibility & Open/Closed principles, the first two principles in the SOLID acronym, are covered in a previous blog post, so we are only examining the final three in this article.

**Liskov Substitution Principle**

“Objects of a superclass shall be replaceable with objects of its subclasses without breaking the application.”

Let’s demonstrate this principle using the Echo Server. 

```
public class EchoServer {
    SocketWrapper socketWrapper;

    public EchoServer(SocketWrapper socketWrapper) {
        this.socketWrapper = socketWrapper;
    }

    public void start(int port) {
        try {
            ServerSocket serverSocket = socketWrapper.startServerSocket(port);
            socketWrapper.connectToClient(serverSocket);
            String clientData;

            while ((clientData = socketWrapper.receiveData()) != null) {
                socketWrapper.sendData(clientData);
                if (Utils.quit(clientData)) {
                    socketWrapper.close();
                }
            }
            socketWrapper.close();
        } catch (IOException e) {
            Utils.error(String.format("Issue trying to listen on port %s", port), e);
        }
    }
}
```
```
public class RunServer {
    public static void main(String[] args) {
        int port = 8080;

        if (args.length >= 1) {
            port = Integer.parseInt(args[0]);
        } else {
            Utils.print("Port not provided. Using port 8080.");
        }
        ServerSocketWrapper serverSocketWrapper = new ServerSocketWrapper();
        EchoServer server = new EchoServer(serverSocketWrapper);
        server.start(port);
    }
}
```

```
public interface SocketWrapper {
    ServerSocket startServerSocket(int port) throws IOException;
    Socket connectToClient(ServerSocket socket) throws IOException;
    String receiveData();
    void sendData(String data);
    void close();
}
```
An interface is a set of abstractions that an implementing class must follow. In this example, `EchoServer` employs the use of the `SocketWrapper` interface so that it can use a wrapper for it's external APIs, Java's `ServerSocket` and `Socket` classes. In doing this, we can test `EchoServer` without having to deal with the behavior or state of the project's external dependencies. 

In our tests, instead of passing in `ServerSocketWrapper` to the `EchoServer`, we can pass in a mock of `ServerSocketWrapper`, which tests that functions are being called and messages are outputted without spinning up any actual sockets. 

```
class EchoServerTest {
    @Test
    public void testServerSocketIsCreated() {
        PrintWriter output = new PrintWriter(new StringWriter(), true);
        BufferedReader input = new BufferedReader(new StringReader("test"));
        MockServerSocketWrapper mockSocketWrapper = new MockServerSocketWrapper(input, output);
        EchoServer echoServer = new EchoServer(mockSocketWrapper);
        echoServer.start(8080);

        assertTrue(mockSocketWrapper.wasStartServerSocketCalled());
    }
}
```
```
public class ServerSocketWrapper implements SocketWrapper {
    private BufferedReader input;
    private PrintWriter output;
    private ServerSocket serverSocket;

    public ServerSocket startServerSocket(int port) throws IOException {
        serverSocket = new ServerSocket(port);
        Utils.print("Listening for connection on port " + port);
        return serverSocket;
    }
    ...
}
```
```
public class MockServerSocketWrapper implements SocketWrapper {
    private BufferedReader input;
    private PrintWriter output;
    private int mockPort;
    private boolean startServerSocketCalled = false;

    public MockServerSocketWrapper(BufferedReader input, PrintWriter output) {
        this.input = input;
        this.output = output;
    }

    public ServerSocket startServerSocket(int port) {
        mockPort = port;
        startServerSocketCalled = true;
        return null;
    }
    ...
}
```
The Liskov Substitution Principle is applicable when there's a supertype-subtype inheritance relationship; the methods defined in the supertype define a contract for the subtypes. In this instance, the `SocketWrapper` interface acts as the supertype and the wrappers are the subtype. SocketWrapper can be replaced by `ServerSocketWrapper` and `MockServerSocketWrapper` because they adhere to the SocketWrapper contract. Meanwhile, `EchoServer` is blissfully unaware of the specific wrapper it receives as long as it's a subtype of `SocketWrapper` (more on this later).


**Interface Segregation Principle**

“Clients should not be forced to depend upon interfaces that they do not use.“

According to the Interface Segreation Principle, declaring methods in an interface that are not necessary to its subtypes leads to a polluted interface. Instead of one fat interface, small interfaces can be created that serve varying subtypes. 

To adhere to the Interface Segregation Principle, `SocketWrapper` must only declare methods that are absolutely necessary to the functionality of the sockets used in its implementation. 

```
public interface SocketWrapper {
    ServerSocket startServerSocket(int port) throws IOException;
    Socket connectToClient(ServerSocket socket) throws IOException;
    String receiveData();
    void sendData(String data);
    void close();
}
```
In order for `EchoServer` to work, a server socket needs to be created (`startServerSocket`). It also needs to listen for and connect to a client (`connectToClient`), as well as receive input and send output (`receiveData` and `sendData`). Finally, it needs to close once the client stops sending messages (`close`).

But let's say we created a separate `EchoClient` class to connect to the `EchoServer` class. We'd need to use an interface to test the sockets within the `EchoClient` class. We have a `SocketWrapper` interface ready, but is it the correct the interface to use? A client wouldn't need either the `startServerSocket` or `connectToClient` methods because the client is not responsible for any of that functionality. A client does not spin up a server socket, it simply to connects to one that is available. Likewise, a client does not connect to another client, it connects to a server. If we we were to use the `SocketWrapper` interface for a client socket wrapper, we'd have to include those methods within the new client wrapper.

```
public class ClientSocketWrapper implements SocketWrapper {
    private BufferedReader input;
    private PrintWriter output;
    private Socket socket;

    public void startClientSocket(InetAddress host, int port) {
        try {
            socket = new Socket(host, port);
            Utils.print("Connection successful using " + host + " on port " + port);
            buildIOStream();
        } catch (IOException e) {
            Utils.error("Could not create client socket ", e);
        }
    }

    public ServerSocket startServerSocket(int port) throws IOException {
        ...
    };
    ...
}
```
In addition to this, `SocketWrapper` would need to be modified to include the methods that the `ClientSocketWrapper` does use, such as `startClientSocket()`.

```
public interface SocketWrapper {
    ServerSocket startServerSocket(int port) throws IOException;
    Socket connectToClient(ServerSocket socket) throws IOException;
    void startClientSocket(InetAddress host, int port) throws IOException;
    String receiveData();
    void sendData(String data);
    void close();
}
```

This, in turn, means that `ServerSocketWrapper` needs to be modified if it wants to continue to adhere to the `SocketWrapper` contract.

```
public class ServerSocketWrapper implements SocketWrapper {
    private BufferedReader input;
    private PrintWriter output;
    private ServerSocket serverSocket;

    public ServerSocket startServerSocket(int port) throws IOException {
        serverSocket = new ServerSocket(port);
        Utils.print("Listening for connection on port " + port);
        return serverSocket;
    }

    public Socket startClientSocket(InetAddress host, int port) throws IOException {
        ...
    }
    ...
}
```
At this stage, we are in full violation of the Interface Segregation principle. `SocketWrapper` includes methods that are not used by `ServerSocketWrapper` and `ClientSocketWrapper`. We're also in violation of the Open/Closed principle because we've had to modify the interface.

The better way to do this is to simply create a new interface for the client socket wrapper and leave `SocketWrapper` as is. 

```
public interface ClientWrapper {
    void startClientSocket(InetAddress host, int port) throws IOException;
}
```

```
public interface SocketWrapper {
    ServerSocket startServerSocket(int port) throws IOException;
    Socket connectToClient(ServerSocket socket) throws IOException;
    String receiveData();
    void sendData(String data);
    void close();
}
```

**Dependency Inversion Principle**

"High level modules should not depend on low level modules; both should depend on abstractions. Abstractions should not depend on details. Details should depend on abstractions."

In following both the Open/Closed the Liskov Substitution principle, our echo server is already adhering to the Dependency Inversion principle. 

Because it's using an interface, `EchoServer` has no knowledge of either `ServerSocketWrapper` or `MockServerSocketWrapper`. It is decoupled from a specific implementation of `SocketWrapper`. It simply needs a subtype of `SocketWrapper` to be created. 

```
public class RunServer {
    public static void main(String[] args) {
        int port = 8080;

        if (args.length >= 1) {
            port = Integer.parseInt(args[0]);
        } else {
            Utils.print("Port not provided. Using port 8080.");
        }
        ServerSocketWrapper serverSocketWrapper = new ServerSocketWrapper();
        EchoServer server = new EchoServer(serverSocketWrapper);
        server.start(port);
    }
}
```

```
public class EchoServer {
    SocketWrapper socketWrapper;

    // EchoServer only interacts with the interface
    public EchoServer(SocketWrapper socketWrapper) {
        this.socketWrapper = socketWrapper;
    }
    ...
}
```
Conversely, as `ServerSocketWrapper` is a subtype of `SocketWrapper`, it is dependent on the interface for its implementation. It must include all of the methods listed in the interface, as well as the implementation details for all of those methods. 

```
public class ServerSocketWrapper implements SocketWrapper {
    private BufferedReader input;
    private PrintWriter output;
    private ServerSocket serverSocket;
    private Socket clientSocket;

    public ServerSocket startServerSocket(int port) throws IOException {
        serverSocket = new ServerSocket(port);
        Utils.print("Listening for connection on port " + port);
        return serverSocket;
    }

    public Socket connectToClient(ServerSocket serverSocket) {
        try {
            clientSocket = serverSocket.accept();
            Utils.print("Connection successful");
            buildIOStream();
        } catch (IOException e) {
            Utils.print("Connection unsuccessful");
        }
        return clientSocket;
    }

    public void buildIOStream() throws IOException {
        createWriter();
        createReader();
    }

    public String receiveData() {
        try {
            String clientInput = input.readLine();
            Utils.print("Client: " + clientInput);
            return clientInput;
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }

    public void sendData(String data) {
        if (!Utils.quit(data)) {
            output.println(data);
        }
    }

    public void close() {
        try {
            input.close();
            output.close();
            clientSocket.close();
            serverSocket.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private void createWriter() throws IOException {
        output = new PrintWriter(clientSocket.getOutputStream(), true);
    }

    private void createReader() throws IOException {
        input = new BufferedReader(new InputStreamReader(clientSocket.getInputStream()));
    }
}
```


I hope this brief overview of the SOLID principles within the world of Echo Servers is helpful in understanding how they help us write cleaner, less decoupled code. 

