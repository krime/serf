# Introduction #

As applications receive data from the network, they may decide to **stop** reading for a while. This may be due to some local constraints on processing the incoming network data.

Currently, the definition for `serf_response_handler_t` states that the response handler must consume **all** data from the network. This design choice was made to avoid deadlocks between an application that needs to process the incoming data and return data to the server. If the application does not consume the incoming data, then it will not provide a response to the server, which then blocks the server.

There are cases, however, where the application is not ready to process the incoming network data. It _will_ get to it eventually, in order to avoid the deadlock, but it may need to throttle the incoming data due to local resource constraints. For example, in Apache Subversion, the incoming data generated new requests to the server; it was observed that tens of **thousands** of requests would get queued up, based on the processing disparity between reading the incoming data and satisfying the resulting requests. Subversion applied a throttle on the processing of the data, to limit the maximum requests queued at any given time.

In an ideal scenario, the application simply stops reading, and the network's TCP window fills, and the server will stop pushing data onto the network, and (thus) blocks the server process. That would be an ideal end-to-end control flow on the data transport. However, if the applications blocks long enough, the server may close the connection (for many reasons, left as an exercise to the reader). The application needs to read "just enough" to avoid the triggered closure on the server.

Reading from the network, but not delivering it to the app (which has signaled "no interest right now"), implies that the data must be held on the client side. This could be in RAM or within a file on the disk. When the application decides to resume reading, it will begin with this content that has been held.

Many applications may need to deal with this scenario, so it would be best for serf to provide this kind of functionality to applications.

# Details #

In `ra_serf`, the network code reads from the server as fast as it arrives. There is no attempt at pushback all the way to the server. All arriving content is saved into memory, or saved to disk when the in-memory size becomes too large.

Lieven noted that it would be ideal to save the **deflated** content. This is a great insight, but also implies ramifications on the network stack. The application could manage the content-saving from within the response handler, but to save the deflated content is somewhere within the bucket chain.

My initial thought is to add an API to the buckets, let's call it `pushback`. Most buckets cannot do anything with this function, but within the bucket stack (say, between the inflate and dechunk buckets) there would be a "content saver" bucket. That saver bucket would hold content in memory, or spill to disk, according to instructions at creation time. The saver would continue to hold content until a bucket `read` arrived. Note that "somehow" the saver bucket needs to be called by serf's network stack to read from its interior bucket (e.g a dechunk, ssl, or socket bucket, typically). This implies some new connection between the network internals and the bucket managing the pushback request.

The debugging infrastructure would need to change, since it attempts to verify that response handlers read until `EAGAIN` is reached. Calling `pushback` would need to be an acceptable alternative.

This would be an acceptable form of managing the incoming content, but we can do much better. Recall that we merely need to keep the connection "active enough". If the client knows enough about the server, it could apply network-level pushback for (say) 55 seconds, then empty the TCP window into local memory, and avoid the server timeout at the 60 second mark.

With enough observation, the serf library could learn the server timeout. If the requests are granular enough, then little information would be lost on a timeout. Of course, an application could also specify the parameters that it would like to apply to the connection to avoid timeouts. serf could then adjust the pollset to read the connection every **N** seconds to keep the connection alive, yet avoid streaming too much content to the client.

Management of the timing is much more involved than a simple "content saver" bucket. That seems like the proper direction to head, but I have no design suggestions at this time.