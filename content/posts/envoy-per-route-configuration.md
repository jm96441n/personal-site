+++
title = "Envoy Per Route Filter Configuration"
date = "2023-09-28T20:01:34-04:00"
author = "John Maguire"
authorTwitter = "" #do not include @
cover = ""
tags = ["envoy"]
keywords = ["envoy", "typedPerFilter"]
description = "Detailing how to provide per route configuration to override listener configuration"
showFullContent = false
+++

When programming envoy via either the xds interface or through a static yaml file allows us to not only provide filter configuration for a listener, which will
apply to all routes attached to that listener, but we can also provide configuration on a per route basis that will
override the filter configuration defined on the listener. The documentation is not super clear on how to do this, so I'm going
to dive in on some examples to show how it's done and the potential pitfalls and footguns of doing so.

## Setup
If you just want to read this post then you're all set to go! Otherwise if you want to follow along you'll need to setup
a few things on your system. You will need docker in order to run the backend service we'll be routing envoy to,
[func-e](https://func-e.io/) to run envoy, curl to make requests and test our setup, and finally the [example repo]()
which has the envoy configuration files as well as a few scripts to get started and test your setup.

## Envoy Filters

Before diving in, let's do a quick refresher on envoy filters. Envoy filters provide a way to extend the functionality
of envoy and perform actions on requests and/or responses; such as applying rate limiting, header manipulation, or
adding jwt authentication (you can see a full list of filters that envoy ships with
[here](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#extension-category-envoy-filters-http)).
These filters can apply to L3/L4 level traffic or L7 level traffic depending on the filter, for example jwt
authentication can only apply to L7 traffic. Filters are typically defined on the listener and are grouped
together into "filter chains", and a chain is chosen based on matching criteria on the request or you can specify a
default chain that is applied when no other chain matches. Like I mentioned earlier, a filter chain will be run for
every route that is attached to the listener.

## Per Route Configuration

Now that last sentence can make filters on listeners seem relatively inflexible for defining potentially different
filters for different routes attached to a listener, say if you wanted to include more strict RBAC (Role Based Access
Control) rules on an admin route than you would on other normal user routes. Without per route configuration you would need to setup a separate
listener for all admin routes, which doesn't sound like a great UX and means keeping two listeners up to date with many
of the same filters defined.

Let's look at an example for configuring separate RBAC rules for an admin route on a listener and walk through the
specific points that we need to include to make this work.


To start we're going to have an envoy configuration that applies an RBAC rule looking for a header matching `role:
user` (this leaves a lot of trust to the client and really should never be used in a real situation, but it will work
for what we're looking to demonstrate here.) In this configuration we set up a backend service to route to (in the
example repo this is the (http-echo)[https://github.com/hashicorp/http-echo] service from Hashicorp) and we configure
the listener to listen on port 10000 with a prefix match on the route so that we match all routes to go to this service.

```yaml
# basic.yaml in the example repo
static_resources:
  listeners:
  - address:
      socket_address:
        address: 0.0.0.0
        port_value: 10000
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: service
              domains:
              - "*"
              routes:
              - match:
                  prefix: "/"
                route:
                  cluster: local_service
          http_filters:
          - name: envoy.filters.http.rbac
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.rbac.v3.RBAC
              matcher:
                matcher_list:
                  matchers:
                  - predicate:
                      single_predicate:
                        input:
                          name: envoy.matching.inputs.request_headers
                          typed_config:
                            "@type": type.googleapis.com/envoy.type.matcher.v3.HttpRequestHeaderMatchInput
                            header_name: role
                        value_match:
                          exact: user
                    on_match:
                      action:
                        name: action
                        typed_config:
                          "@type": type.googleapis.com/envoy.config.rbac.v3.Action
                          name: unauthorized
                          action: ALLOW
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

  clusters:
  - name: local_service
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: local_service
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: 0.0.0.0
                port_value: 8080

```

To run this first let's make sure that our backend service is read to serve requests by running `make setup` from the
example repo. Next we can get envoy up and running with this configuration by running `make basic` from the repo, this
utilizes `func-e` to run envoy for us. Now to check that our configuration is applied correctly we can run a few curl
commands from another terminal:
```bash
# the following will all return a 403
curl localhost:10000
curl -H "role: admin" localhost:10000
curl -H "role: admin" localhost:10000/admin

# the following will all allow the request through
curl -H "role: user" localhost:10000
curl -H "role: user" localhost:10000/admin
```

Now let's expand this a bit to include an additional constraint that when we match the `/admin` route we should check for a header of
`role:admin` and all other routes should continue to function as is. We could set up an entirely different filter chain
to handle this, but that means repeating a bunch of configuration for what is a small override for the single route.

```yaml
# routeConfig.yaml in the example repo
static_resources:
  listeners:
  - address:
      socket_address:
        address: 0.0.0.0
        port_value: 10000
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: service
              domains:
              - "*"
              # this route contains the changes, you can see we use the typed_per_filter_config here
              routes:
              - match:
                  prefix: "/admin"
                route:
                  cluster: local_service
                typed_per_filter_config:
                  envoy.filters.http.rbac:
                    "@type": type.googleapis.com/envoy.extensions.filters.http.rbac.v3.RBACPerRoute
                    rbac:
                      matcher:
                        matcher_list:
                          matchers:
                          - predicate:
                              single_predicate:
                                input:
                                  name: envoy.matching.inputs.request_headers
                                  typed_config:
                                    "@type": type.googleapis.com/envoy.type.matcher.v3.HttpRequestHeaderMatchInput
                                    header_name: role
                                value_match:
                                  exact: admin
                            on_match:
                              action:
                                name: action
                                typed_config:
                                  "@type": type.googleapis.com/envoy.config.rbac.v3.Action
                                  name: unauthorized
                                  action: ALLOW
              - match:
                  prefix: "/"
                route:
                  cluster: local_service
          http_filters:
          - name: envoy.filters.http.rbac
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.rbac.v3.RBAC
              matcher:
                matcher_list:
                  matchers:
                  - predicate:
                      single_predicate:
                        input:
                          name: envoy.matching.inputs.request_headers
                          typed_config:
                            "@type": type.googleapis.com/envoy.type.matcher.v3.HttpRequestHeaderMatchInput
                            header_name: role
                        value_match:
                          exact: user
                    on_match:
                      action:
                        name: action
                        typed_config:
                          "@type": type.googleapis.com/envoy.config.rbac.v3.Action
                          name: unauthorized
                          action: ALLOW
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

  clusters:
  - name: local_service
    type: STRICT_DNS
    lb_policy: ROUND_ROBIN
    load_assignment:
      cluster_name: local_service
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: 0.0.0.0
                port_value: 8080
```

To run this first let's make sure that our backend service is read to serve requests by running `make setup` from the
example repo. Next we can get envoy up and running with this configuration by running `make per-route` from the repo, this
utilizes `func-e` to run envoy for us. Now to check that our configuration is applied correctly we can run a few curl
commands from another terminal window:

```bash
# the following will all return a 403
curl localhost:10000
curl -H "role: admin" localhost:10000
curl -H "role: user" localhost:10000/admin

# the following will all allow the request through
curl -H "role: user" localhost:10000
curl -H "role: admin" localhost:10000/admin
```

So we can see that we've now setup per route overrides for our RBAC configuration! We did this by adding a
`typed_per_filter_config` field to our route configuration for the `/admin` route and specified the envoy rbac filter as
the key and a [RBACPerRoute](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/http/rbac/v3/rbac.proto#envoy-v3-api-msg-extensions-filters-http-rbac-v3-rbacperroute) type.
The `typed_per_filter_config` is a simple map with a key of a string (the name of the filter) and the value is the
actual per route filter to use (you can find the full list [here](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#envoy-v3-api-field-extensions-filters-network-http-connection-manager-v3-httpfilter-name).)

## Footguns

* **All the per route configurations function a little differently**
One thing to keep in mind is that every per route configuration is slightly different, for example when configuring JWT
authentication the listener level filter must have a [map](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/http/jwt_authn/v3/config.proto#extensions-filters-http-jwt-authn-v3-jwtauthentication)
with a key of the name to a value of a JWT configuration that is then referenced by
name in the [PerRouteConfig](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/http/jwt_authn/v3/config.proto#extensions-filters-http-jwt-authn-v3-perrouteconfig).

*  **The per route filters must also be present on the listener**
The other potential footgun here is that the per route filters _will not
apply_ unless there is a a filter of the same type defined on the listener. This means that the key in the
`typed_per_filter` config must match the name of a filter on the listener. In some cases this means defining an empty
configuration for the filter at the listener level, such as if you only want to have RBAC rules enforced on the admin
route but leave all other routes unaffected.

## Wrapping Up

Hopefully this clarifies how to override listener filters to have different filter configuration on a per route basis.
As always feel free to reach out to me via email at [john@jmaguire.tech](mailto:john@jmaguire.tech?subject=[EnvoyPerRouteConfiguration])
