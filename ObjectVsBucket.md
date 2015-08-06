**DRAFT**

This page is a discussion document.


---


# Introduction #

Paul noted there are buckets which process (and expose) information that fall outside of the design principles behind buckets. Buckets are best viewed as filters or transformational systems. The buckets which extract information (usually with some kind of state machine) are the odd-man-out. They expose APIs outside of the typical bucket protocol, which implies they are **not** buckets.

The proposal is to eliminate such buckets, and move the implied objects to a first-class status.

Note that we would like to retain socket-level processing in the control loop. Applications should be allowed to attach to raw data streams. Yet most serf client applications will work at the http-response-callback level instead of the raw network.

# Details #

There are several bucket types that process data outside of the "streamy" flow inherent to the bucket design. Below are some ideas on a strategy to upgrade them to a first-class status, rather than squeezing them into the bucket model.

## Response (`response_buckets.c`) ##

serf should have an internal response handler that processes data from the network stream (e.g. the stack of SSL, deflate, dechunk, etc buckets). That handler should process the headers, then present an application-level callback with a "response object" that contains the status code and reason, all the headers, and a "body bucket" to read the contents of the response.

Applications can wait for a valid object-level response, rather than needing to churn the state machines which process the HTTP response.

## Headers (`headers_buckets.c`) ##

This will be folded into the response processing, described above.