---
layout: post
title:  "On SOLID's Liskov Substitution, Interface Segregation, and Dependency Inversion Principles"
date:   2022-01-19 14:57:41 -0400
categories: blog
---

Writing an Echo Server that follows SOLID principles

**What is an Echo Server?**

An Echo Server is an application that allows a client and a server to establish a connection from which the server can “echo” back messages it receives from the client. The client and server communicate via sockets. Both parties bind sockets to their end of the connection, reading from and writing to the bound socket. 

Writing an Echo Server is a great way to see how the SOLID principles work in practice. 

**SOLID principles**

The SOLID principles are the five basic principles of object oriented programming. Their aim is to make designs easier to understand, maintain, and extend. They do this by reducing dependencies within the code. 

The Single Responsibility & Open/Closed principles, the first two principles in the SOLID acronym, are covered in a previous blog post, so we are only examining the final three in this article.

**Liskov Substitution Principle**

“Objects of a superclass shall be replaceable with objects of its subclasses without breaking the application.”

```
class ExampleClass {
  void exampleMethod(SuperClass superClass) {
    exampleHelperMethod(superClass);
  }
```

According to the Liskov Substitution principle, any subclass object of `SuperClass` should be able to be passed into `exampleMethod` without changing or breaking the application.

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
To make this more clear, here are the `ServerSocketWrapper` and `MockServerSocketWrapper` classes:

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
The Liskov Substitution Principle is applicable when there's a supertype-subtype inheritance relationship; the methods defined in the supertype define a contract for the subtypes. In this instance, the `SocketWrapper` interface acts as the supertype and the wrappers are the subtype. SocketWrapper can be replaced by `ServerSocketWrapper` and `MockServerSocketWrapper` because they adhere to the SocketWrapper contract. Meanwhile, `EchoServer` is blissfully unaware of the specific wrapper it receives as long as it's a subtype of `SocketWrapper`.

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
In order for EchoServer to work, a server socket needs to be created (startServerSocket()). It also needs to listen for and connect to a client (connectToClient()), as well as receive input and send output (receiveData() and sendData()). Finally, it needs to close once the client stops sending messages (close()). All of those methods are needed for the echo server to run from start to finish. 

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
At this stage, we are in full violation of the Interface Segregation principle. Our interface includes methods that are not used by `ServerSocketWrapper` and `ClientSocketWrapper`. We're also in violation of the Open/Closed principle because we've had to modify the `ServerSocketWrapper` to extend the `SocketWrapper` interface.

The better way to do this is to simply create a new interface for the `ClientSocketWrapper` and revert the old interface back to it's original state. 

```
public interface ClientWrapper {
    void startClientSocket(InetAddress host, int port) throws IOException;
    String getUserInput() throws IOException;
    String receiveData();
    void sendData(String data);
    void close();
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
















