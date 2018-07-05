+++
title = "ZeroMQ XPUB sockets"
description = "ZeroMQ XPUB sockets"
date = "2012-04-04"
categories = [ "Development" ]
+++

I was wlaying around with the new [XPUB &
XSUB](http://www.zeromq.org/whitepapers:0mq-3-0-pubsub) ZeroMQ socket types
that allow subscription forwarding, e.g. the producer (XPUB) can track what
values he has to produce, which comes in handy when the value production is an
expensive operation and the producer cannot afford to produce _all_ values.

```python
import thread
import time
import zmq
from zmq.core.socket import Socket

# global zmg context
context = zmq.Context()
endpoint = "tcp://*:8888"

# the subscriber thread function
def subscriber(name, address, cnt, subscriptions):
    print ("starting worker thread %s subscribing to %s for %s"%(name,address,subscriptions))
    sub = Socket(context, zmq.SUB)
    sub.connect(address)
    for subscription in subscriptions:
        sub.setsockopt(zmq.SUBSCRIBE, subscription)
    for x in range(0, cnt):
        print ("%s received %s" % (name, sub.recv()))
    print ("%s closing socket after %d messages" % (name, cnt))
    sub.close()

def main():
    publisher = Socket(context, zmq.XPUB)
    publisher.bind(endpoint)

    address = "tcp://localhost:8888"
    thread.start_new(subscriber, ("s1", address, 10, ["a", "b"]))
    thread.start_new(subscriber, ("s2", address, 20, ["b", "c"]))

    subscriptions = []
    r = 0
    while True:
        # handle subscription flow first to decide what messages need to be produced
        while True:
            try:
                rc = publisher.recv(zmq.NOBLOCK)
                subscription = rc[1:]
                status = rc[0]== "\x01"
                method = subscriptions.append if status else subscriptions.remove
                method(subscription)
            except zmq.ZMQError:
                break

        # produce a value for each existing subscription
        for subscription in subscriptions:
            print "sending " + subscription
            publisher.send("%s %d" % (subscription, r))
        time.sleep(0.5)
        r += 1

if __name__ == "__main__":
    main()
```