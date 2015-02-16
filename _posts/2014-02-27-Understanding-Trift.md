---
layout: post
title:  "Understanding Thrift"
date:   2014-02-27
categories: thrift 
---

In this post, I will explain how Thrift works internally by following the consecutive steps throughout the generated code. The code snippets in this post are based on the generated Thrift code, but are simplified to only show the core functionality.


![alt text](/images/thrift_flow.gif "Thrift Flow")

1. [The first step](#generating_the_java_code) is to generate the Thrift code from the IDL]
2. [The second step](#client_side) goes into detail how the client makes the remote procedure call (RPC) to the server
3. Finally, [the third step](#server_side) explains how the server receives the RPC message and returns the reply.
4. [Additional information](#additional_information) on client and server transport


# Generating the Java code

The [Thrift IDL example](http://thrift.apache.org/tutorial/java/) that comes with Thrift 0.9 defines a Calculator service. I will only focus on the `add` function for the synchronous clients. Additional Thrift features such as remote exceptions, one way calls or async RPC's are easy to understand once the basic  `add` function is understood. 


{% highlight java %}
service Calculator extends shared.SharedService {
   void ping(),
   i32 add(1:i32 num1, 2:i32 num2),
   i32 calculate(1:i32 logid, 2:Work w) throws (1:InvalidOperation ouch),
   oneway void zip()
}
{% endhighlight %}

The Java code that underlies the client and server are generated using:


{% highlight shell %}
thrift -r -gen java tutorial.thrift
{% endhighlight %}

The generated code is located in gen-java. 

#Client side

{% highlight java %}
TTransport transport = new TSocket(host, port);
TProtocol protocol = new TBinaryProtocol();
Calculator.Client client = new Calculator.Client(protocol);
int r = client.add(1,2);
{% endhighlight %}


The constructor of the client passes the TBinaryProtocol. This protocol will be used for all communication (in- and outgoing) between client and server.

{% highlight java %}
Calculator.Client (TProtocol p) {
  super (p);
}
// Calculate.Client extends org.apache.thrift.TServiceClient
TServiceClient (TProtocol p) {
 iprot_ = p; // input protocol
 oprot_ = p; // output protocol
}
{% endhighlight %}

The client makes a RPC to the server using 2 arguments and expects an integer as result.

{% highlight java %}
int r = client.add (1,2);
{% endhighlight %}

The generated code transforms this function call to a sequence of sending and receving information. The main information to be send is the name of the function to be called "
add" and the arguments: 1 and 2.

{% highlight java %}
int Calculator.Client.add (int num1, int num2) {
  send_add (i1,i2);
  return recv_add ();
}

void Calculator.Client.send_add (int num1, int num2) {
  add_args a = new add_args ();
  a.setNum1 (num1);
  a.setNum2 (num2);
  sendBase ("add", a);
}
{% endhighlight %}

The arguments (similar for the results) for each services are wrapped into a Java class. The class add_args (derived from TBase) is a placeholder for the arguments of the RPC.
The function sendBase is implemented by the TServiceClient, the parent of Calculator.Client

{% highlight java %}
void TServiceClient.sendBase (String name, TBase args) {
  oprot_.writeMessageBegin (new TMessage(methodName, TMessageType.CALL, ++seqid));
  args.write (oprot_);
  oprot_.writeMessagEnd ();
  oprot_.getTransport ().flush ();
}
{% endhighlight %}

The sendBase writes the header of the RPC to the protocol, then the instance of the argument class (add_args) writes the values (1 and 2) to the TProtocol, the parent class writes the tail of the RPC and finally, the tranport layer is called to send the message to the server.

As mentioned before, the argument class add_args is a placeholder for the arguments of the RPC. In this case "add", it holds 2 integers. The class has convenience methods, such as getters, setters, and deals with optional fields etc. The main functionality of the class is to write the values of the arguments to a TProtocol. Since a TProtocol supports different schemes (StandardScheme and TupleScheme), the argument class must implement a read/write for each scheme

{% highlight java %}
class add_args implements org.apache.thrift.TBase<...>, ... {
public void write (org.apache.thrift.protocol.TProtocol oprot) ... {
  /* according to the scheme defined in iprot, 
   * use add_argsStandardSchemeFactory or add_argsTupleSchemeFactory 
   * to write the values to the protocol */
 } 
}

class add_argsStandardScheme extends StandardScheme<..> {
  
  public void write (TProtocol oprot, add_args struct) {
    oprot.writeStructBegin (STRUCT_DESC);
    oprot.writeFieldBegin (new TField ("num1", TType.I32, 1));
    oprot.writeI32 (struct.num1);
    oprot.writeFieldEnd ();
    oprot.writeFieldBegin (new TField ("num2", TType.I32, 2));
    oprot.writeI32 (struct.num2);
    oprot.writeFieldEnd ();
    oprot.writeStructEnd ();
  }
}
// different way of encoding the message
class add_argsTupleScheme extends TupleScheme<..> {
  public void write (TProtocol oprot, add_args struct) {
    ..
  }
}
{% endhighlight %}
 
In this example, the concrete TProtocol used is TBinaryProtocol. Each TProtocol is associated with a TTransport. 


{% highlight java %}
class TBinaryProtocol ... extends TProtocol {
  public TBinaryProtocol(TTransport trans) { 
    super(trans) 
  }
  public void writeMessageBegin(TMessage message) .. {
    writeString(message.name);
    writeByte(message.type);
    writeI32(message.seqid);
  }
  public void writeString(String s) .. {
    byte[] dat = s.getBytes("UTF-8");
    writeI32(s.length)
    transport_.write(s,0,s.length);
  }
}

class TProtocol ... {
  public void TProtocol.writeI32( int i) ... {
  bytes[0] = (byte)(0xff & (i >> 24));
  bytes[1] = (byte)(0xff & (i >> 16));
  ..
  transport_.write(bytes,0,4);
}
{% endhighlight %}

The TTransport is implemented by TSocket. In this example, the TTransport is implemented by TSocket. This class communicates over sockets and uses standard Java IO streams. 

{% highlight java %}
TSocket(Socket socket) {
  inputStream_ = new BufferedInputStream(socket.getInputStream(), ... );
  outputStream_ = new BufferedOutputStream(socket.getOutputStream(), ... );
}

TSocket.write(byte[] buf, int offset, int len) .. {
  outputStream_.write(buf,offset,len);
}
{% endhighlight %}



# Server side

The server side of a Thrift code base consists of 2 components:
1. The handler which implements the actual service (ping, add, ...)
2. The server which takes care of the communication with the different clients.

## The handler

The handler is the only class that needs to be coded by the end user. This class implements the services defined in the thrift IDL. These services are part of an interface : 

{% highlight java %}
public interface Calculator.Iface {
  public void ping();
  public int add(int num1, int num2);
  ...
}

public class CalculatorHandler implements Calculator.Iface {
  void ping() {
  }

  int add(int num1, int num2) {
    return num1+ num2;
}
{% endhighlight %}


## The server

The server connects the handler, the processor, the protocols and the transports. 

{% highlight java %}
CalculatorHandler handler = new CalculatorHandle();
Calculator.Processor processor = new Calculator.Processor(handler);
{% endhighlight %}

The Processor exposes a map of names associated with functions (ProcessFunction). Each name corresponds to the name of a function ("ping", "add", ...) and the associated functions will wrap the call to the appropriate service in the handler.

{% highlight java %}
Calculator.Processor(Calculator.Iface handler, ... ) {
  map.put("ping", new ping());
  map.put("add", new add()); 
  ...
}
{% endhighlight %}

Inside the Calculator.Processor, there is a class for each service (ProcessFunction). Each class implements the getResult function which is responsible for calling the service in the handler.

{% highlight java %}
class Calculator.Processor.ping<...> {
  ..
  void getResult(Calculator.Iface handler, ping_args args) .. {
    ping_result r = new ping_result();
    handler.ping();
    return r;
  }
}

class Calculator.Processor.add<...> {
  ..
  add_result getResult(Calculator.Iface handler, add_args args) .. {
    add_result r = new add_result();
    r.success = handler.add(args.num1, args.num2);
    r.setSuccessIsSet(true);
    return r;
  }
}
{% endhighlight %}

The TServerSocket creates an instance of a standard Java server socket.

{% highlight java %}
TServerTransport serverTransport = new TServerSocket(9090);
{% endhighlight %}

The TSimpleServer starts listening to the TServerSocket. 

{% highlight java %}
TServer server = new TSimpleServer(new Args(serverTransport).processor(processor));
server.serve();
{% endhighlight %}

The server waits for incoming messages. The message is decoded, the appropriate services in the handler are called and the result is returned. Building the results follows the same logics as building the arguments (add_args and add_result classes).


{% highlight java %}
public void TServer.serve() ... {
 
  TTransport client = serverTransport_.accept();
  if (client != null) {
    TProcessor processor = processorFactory_.getProcessor(client); 
    // the processor c'ted by main, in other words, an instance of Calculator.Processor

    TTransport inputTransport = inputTransportFactory_.getTransport(client); 
    // TServerSocket

    TTransport outputTransport = outputTransportFactory_.getTransport(client); 
    TProtocol iprot = inputProtocolFactory_.getProtocol(client);
    TProtocol oprot = outputProtocolFactory_.getProtocol(client);
    ...
    processor.process(iprot, oprot);
    ...
}
{% endhighlight %}

The dispatching of the incoming message to the service in the handler is done by the Processor.

{% highlight java %}
Calculator.Processor extends org.apache.thrift.TBaseProcessor<..> implements org.apache.thrift.Processor 

Calculator.Processor contains a pointer to the handler

TBaseProcessor.process(TProtocol i, TProtocol o) {
  TMessage msg = in.readMessageBegin();

  ProcessFunction fn = processMap.get(msg.name); // retrieve the name of the RPC and retrieve the associated function
  fn.process(msg.seqId, in, out, handler);
  return true;
}
{% endhighlight %}

For the function `add`, fn is an instance of type `Calculator.Processor.add<...>` which extends the `ProcessFunction`.

The process function is templated with the handler class (CalculatorHandler) and the argument class (Calculator.add_args)

{% highlight java %}
ProcessorFunction<Iface, T extends TBase> {
  process(int seqId, TProtocol iprot, TProtocol oprot, Iface handler) {
    T args = getEmptyArgs()
    args.read(iprot);  // the add_args read from the input stream and populates num1 and num2
    iprot.readMessageEnd();

    TBase result = getResult(iface, args); 

    oprot.writeMessageBegin(new TMessage("add", TMessageType.REPLY, seqid));
    result.write(oprot);
    oprot.writeMessageEnd();
    oprot.getTransport().flush();
  } 
}
{% endhighlight %}

# Additional Information

## Server 

The server binds the TProtocol and the TTransport together: it listens to incoming messages using the choosen protocol and passes the message to the processor.

* TSimpleServer: single threaded, blocking IO server
* TThreadPoolServer: multi threaded, blocking IO server
* TNonblockingServer: multi threaded, non-blocking IO server

## TTransport

The transport used on the client side must correspond to the one used on the server side. The following scheme is limited to socket based transports.

* TSocket: simple socket communication
* TSSLTransport: secure socket communication
* TSaslTransport: [simple authentication and security layer](http://en.wikipedia.org/wiki/Simple_Authentication_and_Security_Layer). The SASL mechanism supports a series of challenges& responses, such as ANONYMOUS, PLAIN, DIGEST-MD5 and GSSAPI. The GSSAPI supports Kerberos [Here is an example](http://stackoverflow.com/questions/13803354/kerberos-for-thrift). For the Java world, the javax.security.sasl module is used, I haven't found a Thrift C/C++ SASL client.

![alt text](/images/thrift_transport.gif "Thrift TTransport")
