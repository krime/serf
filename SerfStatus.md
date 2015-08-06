  * **Site** : http://code.google.com/p/serf/
  * **Code**  : http://serf.googlecode.com/svn/
  * **Issues**: http://code.google.com/p/serf/issues/list
  * **Mail** : [serf-dev@googlegroups.com](http://groups-beta.google.com/group/serf-dev)
  * **People** : Justin Erenkrantz, Greg Stein

# Release Showstoppers #

None at this time.

# Release Nice-to-haves #

### Polish ###

  * Publish doxygen docs on website.

# Version 2.0 #

### Switch to SCons ###

_more info goes here_

See [open issues](http://code.google.com/p/serf/issues/list?can=2&q=label%3Ascons) that should be considered during the SCons work.

### Switch to PoCore ###

_more info goes here_

See [PoCore](http://code.google.com/p/pocore/)

### Connection Types ###

_more info goes here_

### Request Manager ###

_more info goes here_

# NEEDS UPDATE #

Pretty much everything below here needs a review, update, and/or deletion.


---


# Open Issues #

## A better way to control debug checks ##

They are too expensive to do in a release build by default and changes
the external ABI.  We currently disable them by default and require
manual editing of serf.h to enable them.

## Stream Metadata ##

Specifying metadata that describes the stream rather than being
associated with a single bucket type.

Currently, each bucket may define a custom type that allows a bucket
to have some attached metadata.  See
`serf_bucket_request_get_headers()`, etc.

The original idea for metadata was to have a way to specify things
like content type, charset, or the content encoding (for the data
residing in the bucket).

I (Greg) would propose that in tossing the metadata, we create a
"`serf_content_description`" type which can describe the data within a
bucket. These descriptions would be at least a few pointers and might
be too much overhead per bucket. Thus, a solution would be a registry
of content descriptions. A bucket can simply refer to a descrption
stored in the registry. (maybe some common, static descriptions plus
the dynamic registry) Unknown how a dynamic registry would be
built. We could make it an app-provided piece, or we could
automatically create and associate one with a `serf_context`. I'd go
with an app thingy since the registry could be shared across
multiple contexts.

## Null OUT parameters ##

We need to review the various OUT parameters and see if any make sense
to allow a NULL value to designate non-interest in the result.

Also review functions which **don't** return an `apr_status` to see if
they _should_.

## Misc small issues ##

  * Produce binaries for Windows users?  Will Subversion do this for free?
  * Go through design-guide.txt; create a detailed 'walking tour' of serf.
  * Add a higher-level, more dev-friendly API.
    * See Subversion's `ra_serf` for an example of how this might be done.
    * Add an Authentication Abstraction/Utility functions
      * Support for Digest
  * Add a bit of polish to serfmake script.

# Issues on the backburner (for now) #

## Replacing serf\_bucket\_alloc\_t ##

Rather than having `serf_bucket_alloc_t`, can we make
`apr_allocator_t` work "well" for bunches o' small allocations?

Justin says:

> Take the 'freelist' code from apr-util's bucket allocator and merge
> that (somehow) with `apr_allocator_t`.
> (< `MIN_ALLOC_SIZE` allocations - i.e. <8k)

Sander says:

> The cost-per-alloc'd-byte is too expensive. the `apr_allocator`
> works best for 4k multiples. `apr_allocator` provides a buffer
> between system alloc and the app. small allocs (such as what serf
> likes) should be built as another layer.

Justin says:

> So, let's put this on hold...

## Rethink memory usage ##

Memory usage probably needs some thought. in particular, the mix
between the bucket allocators and any pools associated with the
connection and the responses. see the point below about buckets
needing pools.

Justin says:

> Oh, geez, this is awful.  Really awful.  pools are just going to be
> abused.  For now, I've tied a pool with the
> `serf_bucket_allocator` as I can't think of a better short-term
> solution.  I think we can revisit this once we have a better idea
> how the buckets are going to operate.  I'd like to get a better idea
> how filters are going to be written before deciding upon our pool
> usage.

Greg says:

> I think an allocator will be used for for a whole context's worth of
> buckets. any pool within this system ought to be transaction-scoped
> (e.g. tied to a particular request/response pair).

Justin says:

> The change to delay creation of the request/response pool until we
> are about to deliver the request to the server is about the best
> that we can do.  So, we'll have as many 8k-min pools open as there
> are requests in-flight. I'm not sure we can really do better than
> that (and still use pools).

## Allocators in serf\_request\_t ##

`serf_request_t` should contain two allocators: one for request
buckets and one for response buckets. When the context loop is
(re)entered, we can call `serf_debug__entered_loop` on the
response buckets. That will ensure that all responses were fully
drained.

The allocator for the response buckets should be passed to the
acceptor callback. Today, acceptors use `serf_request_get_alloc` to get
an allocator. We can just pass the "right" allocator.

`serf_request_get_alloc` will return the "`request allocator`"
to be used for allocating all buckets that go into the request.

Note that the app will still retain an allocator for buckets related
to the connection (such as socket buckets created in the connection
setup callback).

## UNDECIDED: Per-connection allocator ##

We should probably create an allocator per connection (within the pool
passed to `connection_create`). That allocator can then be used by a
default connection setup callback.

We could instead create an allocator per context. It probably isn't
important to make this allocator too fine-grained since
connection-level buckets will typically be handled by serf itself and
it will manage them properly.

Maybe pass this allocator to the connection setup callback, similar to
the acceptor. "put the stream bucket in **this** allocator."

## Future Work ##

SPDY -- http://dev.chromium.org/spdy/spdy-whitepaper