# HTTP Caching
#public/github
[HTTP caching - HTTP | MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching)

* caching stores a resource for later use which avoids reaching out to the originating server to fetch the resource
* two main categories of caching - private and shared
	* A_shared cache_is a cache that stores responses for reuse by more than one user
	* A_private cache_is dedicated to a single user![](HTTP%20Caching/HTTPCachtType.png)
* private caching is like your browser cache. It holds documents downloaded via HTTP so you can easily navigate back to them without having to hit the server again
* an example of a shared cached might be your ISP having a web proxy in its local network that serves many users popular resources
* caching usually limited to GETs
* common cache responses:
	* 200 (ok) - successful retrieval of resource
	* 301 (moved permanently) 
	* 404
	* 206 (partial content) for incomplete results

## The `Cache-control` header
* used to specify directives for caching requests and responses

### No cache storage
to indicate that the cache should not store anything about the request or response. Full download on each response
```
Cache-Control: no-store
Cache-Control: no-cache, no-store, must-revalidate
```

### No caching
a cache will send the request to the server for validation before releasing a cached copy
```
Cache-control: no-cache
```

### Private and Public caches
“public” the response may be cached by any cache. I can be useful if pages with HTTP authentication or normally non-cacheable response codes should be cached
```
Cache-control: public
```

“private” indicates the response is for a single user only and shouldn’t be stored by a shared cache
```
Cache-control: private
```

### Expiration
use “max-age=“ to indicate how long a resource should be considered fresh. This is relative to the time of the request, which differs from `Expires`
```
Cache-control: max-age=31536000
```

### Validation
using “must-revalidate” means the cache must verify the status of stale resources before using it
```
Cache-control: must-revalidate
```

## Freshness
_cache-eviction_: periodically removing items from the cache because caches have finite storage

* with expiration times, when a resource is fetched before that time, it is _fresh_, and afterwards, it is _stale_
* eviction algorithms usually prioritize fresh resources
* stale resources are not evicted or ignored, when the cache receives a request for a stale resource, it forwards the request with a `If-None-Match` header to check if it is still fresh
	* if fresh, the server returns a 304 (not modified)
* If the `Cache-control: max-age=` header is not specified, the `Expires` header will be checked.
	* Expires is an http-date timestamp
	* `Expires: Wed, 21 Oct 2015 07:28:00 GMT`

## Cache validation
* revalidation is triggered when a user presses the reload button, or if `Cache-control: must-revalidate` is passed
* When a cached document’s expiration time has been reached, it is either validated or fetched again
	* it can only be validated if the server provided a validator - be it _strong_ or _weak_

### ETags
	* The ETag response header can be used as a strong validator
	* The HTTP user-agent (like a browser) doesn’t know what this string represents and can’t predict its value
	* If an ETag was included in the response, the client can use it in subsequent requests
		* It can be passed with the `If-None-Match` header
```
If-None-Match: "bfc13a64729c4290ef5b2c2730249c88ca92d82d"
```
* The `Last-Modified` header can be used as a weak validator (only has a 1-second resolution)
	* If `Last-Modified` is included in the response, subsequent requests can include the `If-Modified-Since` header to validate the cached document
	* When a validation request is made, the server can either ignore the validation request and return a 200, or return a 304 to indicate that the cached value was used

## Varying Responses
* The `Vary` response header is used to determine how to match future request headers to decide wether or not to use a cached response 
* A cached response will only be used if all the conditions of the `Vary` header have been met
* i.e. using `Vary: User-Agent` will indicate to caching servers that they should consider the User-Agent when deciding whether to serve the page from a cache. 
	* This would be useful if you served different content to mobile vs. desktop users
