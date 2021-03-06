---
layout: post
title: Receiving Datagrams
tags: [C++, network]
---

The [socket API](https://en.wikipedia.org/wiki/Berkeley_sockets) is defined such that the user supplies a buffer of a pre-determined size that UDP datagrams are read into. If the buffer is smaller than the datagram, then the surplus bytes from the datagram are discarded.

How can we read the entire datagram when we do not know its size in advance?

<!--more-->

## Introduction

One approach is to define the maximum datagram size at the application protocol level. This approach is fine if you control both the sending and the receiving end.

Another approach is to use the [maximum transmission unit](https://en.wikipedia.org/wiki/Maximum_transmission_unit) minus header overhead as the datagram size. However, the sender can still send larger datagrams, in which case the IP stack will break them into smaller [fragments](https://en.wikipedia.org/wiki/IP_fragmentation) and reassemble them on the receiving side before passing the resulting datagram on to the user.

A third approach is to get the datagram size from the IP stack before reading it. We will describe this approach below with example code written in [Boost.Asio](http://www.boost.org/doc/html/asio.html).

We start with synchronous case, and then proceed to the asynchronous case.

## Synchronous reading

The basic idea is to query the size of the next datagram from the UDP socket using the [`bytes_readable`](http://www.boost.org/doc/html/boost_asio/reference/basic_datagram_socket/bytes_readable.html) attribute. Obviously we cannot query the size of the next datagram before it has arrived, so first we have to wait for it to arrive.

The standard trick is to use [`null_buffers`](http://www.boost.org/doc/html/boost_asio/reference/null_buffers.html), which is a special buffer type that is used to indicate that we only want to wait for data without actually reading it. Notice that we ignore the return value, because it is always zero.

{% highlight c++ %}
// Wait for data to arrive on socket
socket.receive_from(boost::asio::null_buffers(),
                    remote_endpoint);
{% endhighlight %}

While the above works, it does not tell us what remote endpoint the datagram has been sent from. We could wait by [peeking](http://www.boost.org/doc/html/boost_asio/reference/socket_base/message_peek.html) instead, if we need the remote endpoint before reading the datagram.

{% highlight c++ %}
// Wait for data to arrive on socket
socket.receive_from(boost::asio::buffer(boost::asio::mutable_buffer()),
                    remote_endpoint,
		    decltype(socket)::message_peek);
{% endhighlight %}

We pass a default-constructed [`mutable_buffer`](http://www.boost.org/doc/html/boost_asio/reference/mutable_buffer.html), which is a zero-sized buffer. This will only fill in the remote endpoint.

Normally we should keep the buffer alive until the call is completed because [`buffer`](http://www.boost.org/doc/html/boost_asio/reference/buffer.html) is just a view of our buffer, so passing a view of temporary variable is a generally bad idea -- this is not really important in the synchronous case, but in the asynchronous case it will be. In this particular case the operation is safe because the view stores the data pointer and the size, which are `{nullptr, 0}` here.

Now we can query the size of the next datagram.
{% highlight c++ %}
// Get datagram size
decltype(socket)::bytes_readable readable(true);
socket.io_control(readable);
auto length = readable.get();
{% endhighlight %}

And finally we can allocate a buffer of the correct size, and read the full datagram.
{% highlight c++ %}
// Read the datagram
buffer.resize(length); // Assume buffer is a std::vector<char>
socket.receive_from(boost::asio::buffer(buffer.data(), buffer.size()),
                    remote_endpoint);
{% endhighlight %}

## Asynchronous reading

The asynchronous case follows the same pattern.

We are going to start with an asynchronous wait. When the next datagram is available, then we can read it synchronously. Otherwise, the main difference is that we have to handle error codes explicitly to avoid throwing exceptions from an asynchronous operation.

In the example below we assume that `socket` is a class member variable, and that the class inherits from `enable_shared_from_this`. We wait using null buffers, but we could just as well wait using peeking.

{% highlight c++ %}
// Extend life-time of object by turning it into a shared_ptr.
auto self = shared_from_this();

// Wait for data to arrive on socket
socket.async_receive_from(
    boost::asio::null_buffers(),
    remote_endpoint,
    [this, self] (error_code error, std::size_t /* ignore */)
    {
        if (!error)
        {
            // A datagram is available, so we can use synchronous
	    // operations without blocking.

            // Get datagram size
            decltype(socket)::bytes_readable readable(true);
            socket.io_control(readable, error);
            if (!error)
            {
                auto length = readable.get();

                // Read the datagram
                buffer.resize(length);
                socket.receive_from(boost::asio::buffer(buffer.data(),
                                                        buffer.size()),
                                    remote_endpoint,
                                    0, // No message flags
                                    error);
                if (!error)
                {
                    // Operation succeeded
                    return;
                }
            }
        }
        // Operation failed
    });
{% endhighlight %}
