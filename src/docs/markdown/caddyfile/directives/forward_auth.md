---
title: forward_auth (Caddyfile directive)
---

# forward_auth

An opinionated directive which proxies a clone of the request to an authentication gateway, which can decide whether handling should continue, or needs to be sent to a login page.

- [Syntax](#syntax)
- [Expanded Form](#expanded-form)
- [Examples](#examples)

Caddy's [`reverse_proxy`](/docs/caddyfile/directives/reverse_proxy) is capable of performing "pre-check requests" to an external service, but this directive is tailored specifically for the authentication usecase. This directive is actually just a convenient way to use a longer, more common configuration (below).

This directive makes a `GET` request to the configured upstream with the `uri` rewritten:
- If the upstream responds with a `2xx` status code, then access is granted and the header fields in `copy_headers` are copied to the original request, and handling continues.
- Otherwise, if the upstream responds with any other status code, then the upstream's response is copied back to the client. This response should typically involve a redirect to login page of the authentication gateway.

If this behaviour is not exactly what you want, you may take the [expanded form](#expanded-form) below as a basis and customize it to your needs.

All the subdirectives of [`reverse_proxy`](/docs/caddyfile/directives/reverse_proxy) are supported, and passed through to the underlying `reverse_proxy` handler.


## Syntax

```caddy-d
forward_auth [<matcher>] [<upstreams...>] {
	uri          <to>
	copy_headers <fields...>
}
```

- **&lt;upstreams...&gt;** is a list of upstreams (backends) to which to send auth requests.
- **uri** is the URI (path and query) to set on the request sent to the upstream. This will usually be the verification endpoint of the authentication gateway.
- **copy_headers** is a list of HTTP header fields to copy from the response to the original request, when the request has a success status code.

Since this directive is an opinionated wrapper over a reverse proxy, you can use any of [`reverse_proxy`](/docs/caddyfile/directives/reverse_proxy#syntax)'s subdirectives to customize it.


## Expanded form

The `forward_auth` directive is the same as the following configuration. Auth gateways like [Authelia](https://www.authelia.com/) work well with this preset. If yours does not, feel free to borrow from this and customize it as needed instead of using the `forward_auth` shortcut.

```caddy-d
reverse_proxy <upstreams...> {
	# Always GET, so that the incoming
	# request's body is not consumed
	method GET

	# Change the URI to the auth gateway's
	# verification endpoint
	rewrite <to>

	# Forward the original method and URI,
	# since they get rewritten above; this
	# is in addition to other X-Forwarded-*
	# headers already set by reverse_proxy
	header_up X-Forwarded-Method {method}
	header_up X-Forwarded-Uri {uri}

	# On a successful response, copy response headers
	@good status 2xx
	handle_response @good {
		request_header {
			# for example, for each copy_headers field...
			Remote-User {rp.header.Remote-User}
			Remote-Email {rp.header.Remote-Email}
		}
	}

	# On a failed response, copy the response back to
	# the client, with some hop-by-hop headers removed
	handle_response {
		copy_response_headers {
			exclude Connection Keep-Alive Te Trailers Transfer-Encoding Upgrade
		}
		copy_response
	}
}
```

## Examples

Delegating authentication to [Authelia](https://www.authelia.com/), before serving your app via a reverse proxy:

```caddy
# Serve the authentication gateway itself
auth.example.com {
	reverse_proxy authelia:9091
}

# Serve your app
app1.example.com {
	forward_auth authelia:9091 {
		uri /api/verify?rd=https://auth.example.com
		copy_headers Remote-User Remote-Groups Remote-Name Remote-Email
	}

	reverse_proxy app1:8080
}
```
