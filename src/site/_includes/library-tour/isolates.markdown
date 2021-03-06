All code in Dart runs in the context of an isolate.
Use additional isolates for concurrent programming
and to run third-party code more securely.

### Isolate concepts

Each isolate has its own heap, which means that all its values in memory,
including globals, are available only to that isolate. The only mechanism
available to communicate between isolates is to pass messages.
Messages are sent through ports.

The content of a message can be:

* A primitive value (null, num, bool, double, String)
* An instance of SendPort
* A list or map whose elements are any of the above, including other lists and maps
* In [special circumstances](#sending-any-type-of-object), an object of any type

Lists and maps can have cyclic references to themselves.
Each message is copied when sent to another isolate, ensuring
one isolate cannot directly change the state in another isolate.

Isolates might run in a separate process or thread, depending on the
implementation. For web applications, isolates can
be compiled to Web workers, if they are available.

### Isolates in the VM

In the standalone VM, the main() function runs in the first isolate
(also known as the _main isolate_).
When the main isolate terminates, it terminates the whole VM, regardless
of whether other isolates are still running.

In the VM, an isolate continues to run as long as it has an open
ReceivePort inside the isolate. If the main function only starts other
isolates, doing no work itself, you must keep the main isolate
alive to keep the program alive. To keep a main isolate alive,
open a ReceivePort in the main isolate.

When all the child isolates have finished their work, you can send a message
to the main isolate to close its ReceivePort, thus stopping the program.

### Using isolates

To use an isolate, you import the isolates library,
spawn a new isolate, and then send and receive messages.

#### Importing the isolates library

The isolates API is in the `dart:isolate` library.

{% highlight dart %}
#import('dart:isolate');

main() {
  // The app
}
{% endhighlight %}

#### Spawning isolates

Any top-level function or static method
([not currently working](#isolates-static-method))
is a valid entry point for an isolate.
The entry point should not expect arguments and should return void.
It is illegal to use a function closure as an entry point to an isolate.
Pass the entry point to
[spawnFunction()](http://api.dartlang.org/dart_isolate.html#spawnFunction).

<aside id="isolates-static-method">
  <div class="alert alert-info">
    <strong>Tip:</strong>
    <b>Note:</b> Both the dart2js compiler and the Dart VM
    <a href="http://code.google.com/p/dart/issues/detail?id=3011">bug #3011</a>
    do not yet support static methods as isolate entry points.
  </div>
</aside>

{% highlight dart %}
#import('dart:isolate');

runInIsolate() {
  print("hello from an isolate!");
}

main() {
  spawnFunction(runInIsolate);

  // Note: incomplete.
  // Use a ReceivePort (details below) to keep the main isolate alive
  // long enough for runInIsolate() to perform its work.
}
{% endhighlight %}

We plan to support spawning an isolate from code at a URI.

#### Sending messages

Send a message to an isolate via a
[SendPort](http://api.dartlang.org/dart_isolate/SendPort.html).
The spawnFunction() method returns a handle to the
newly created isolate's SendPort.

To simply send a message, use
[send()](http://api.dartlang.org/dart_isolate/SendPort.html#send).

{% highlight dart %}
#import('dart:isolate');

echo() {
  // Receive messages here. (See the next section.)
}

main() {
  var sendPort = spawnFunction(echo);
  sendPort.send("Hello from main");

  // Note: incomplete.
  // Use a ReceivePort (details below) to keep the main isolate alive
  // long enough for echo() to perform its work.
}
{% endhighlight %}

#### Sending any type of object

In special circumstances
(such as when using spawnFunction() inside the Dart VM),
it is possible to send any type of object to an isolate.
The object message is copied when sent.

Support for sending arbitrary objects to an isolate
is not yet available when compiling to JavaScript.

#### Receiving messages

Use a [ReceivePort](http://api.dartlang.org/dart_isolate/ReceivePort.html)
to receive messages sent to an isolate. Obtain a handle to the
default ReceivePort
from the top-level [port](http://api.dartlang.org/dart_isolate.html#get:port)
property. You can also create new instances of ReceivePort, if
you want to route messages to different ports and callbacks.

Handle an incoming message with a
callback function passed to the
[receive()](http://api.dartlang.org/dart_isolate/ReceivePort.html#receive)
method.

{% highlight dart %}
#import('dart:isolate');

echo() {
  port.receive((msg, reply) {
    print("I received: $msg");
  });
}

main() {
  var sendPort = spawnFunction(echo);
  sendPort.send("Hello from main");

  // Note: incomplete.
  // Use a ReceivePort (details below) to keep the main isolate alive
  // long enough for runInIsolate() to perform its work.
}
{% endhighlight %}

#### Receiving replies

Use the [call()](http://api.dartlang.org/dart_isolate/SendPort.html#call)
method on SendPort as a simple way to send a
message and receive a reply. The call() method returns a
[Future](http://api.dartlang.org/dart_core/Future.html) for the reply.

{% highlight dart %}
#import('dart:isolate');

echo() {
  port.receive((msg, reply) {
    reply.send("I received: $msg");
  });
}

main() {
  var sendPort = spawnFunction(echo);
  sendPort.call("Hello from main").then((reply) {
    print(reply);    // I received: Hello from main
  });
}
{% endhighlight %}

The call() method creates a new receive port and then sends a message to the
send port with replyTo set to the receive port. When a reply is
received, the receive port closes, and the returned Future completes.

#### Keeping the main isolate alive

As mentioned above, when the main isolate terminates, the entire VM
terminates (even if other child isolates are running). To keep
running, the main isolate must create an open ReceivePort. Before the
main isolate can terminate, the ReceivePort must be closed.

You can coordinate isolates with message passing,
sending a message to inform the main isolate when a child
isolate finishes. Here is an example:

{% highlight dart %}
#import('dart:isolate');

childIsolate() {
  port.receive((msg, replyTo) {
    print('doing some work');
    if (replyTo != null) replyTo.send("shutdown");
  });
}

main() {
  var sender = spawnFunction(childIsolate);
  var receiver = new ReceivePort();
  receiver.receive((msg, _) {
    if (msg == 'shutdown') {
      print('shutting down');
      receiver.close();
    }
  });
  sender.send("do work please", receiver.toSendPort());
}
{% endhighlight %}

In the above example, the child isolate runs to completion
because the main isolate keeps a ReceivePort open.
The main isolate creates a ReceivePort to wait for
a "shutdown" message. The term "shutdown" is arbitrary; the ReceivePort
simply needs to wait for some signal.

Once the main isolate receives a "shutdown" message, it closes the
ReceivePort. With the ReceivePort closed and nothing else to do,
the main isolate terminates, causing the app to exit.
