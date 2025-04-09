# Meta
[meta]: #meta
- Name: Hash-based routing
- Start Date: 2025-04-07
- Author(s): b1tamara
- Status: Draft <!-- Acceptable values: Draft, Approved, On Hold, Superseded -->
- RFC Pull Request: (fill in with PR link after you submit it)


## Summary

Cloud Foundry uses `round-robin` and `least-connection` algorithms for load balancing between Gorouters and backends, but 
they are not suitable for certain use cases, prompting a proposal to introduce hash-based routing as an application
per-route load balancing option to enhance load distribution in specific scenarios.

## Problem

Cloud Foundry currently offers two load balancing algorithms to manage request distribution between Gorouters and 
backends. The `round-robin` algorithm ensures an even distribution of load across all available backends, while 
the `least-connection` algorithm optimizes resource utilization by directing traffic to the backend with the fewest 
active connections. A recent enhancement allows usage of these load balancing algorithms on application route level.

However, these existing algorithms are not ideal for scenarios that require routing based on tenant or other specific 
identifiers.This limitation represents a gap in achieving optimal load distribution tailored to hash-based needs.

## Proposal

We propose introducing hash-based routing as a load balancing algorithm to be used on a per-route basis.
<!-- TODO describe the use case more concrete -->


### Limitations

#### Only Application Per-Route Load Balancing
The hash-based load balancing will be configured exclusively as an application per-route option and will not be available as a global setting.

#### Consistent Hashing
The implementation will support only consistent hashing. A consistent hashing minimizes the need for rehashing, particularly when
the number of application instances varies over the time. As a possible solution, [Maglev hashing](https://storage.googleapis.com/gweb-research2023-media/pubtools/2904.pdf) can be considered.

#### Balance Factor
The hash-based load distribution should consider balance factor to ensure that no single application instance gets overwhelmed 
with requests, even when certain hash buckets attract significantly more traffic than others, e.g. with a balance factor 
of 150, no application instance should exceed 150% of the average load across all instances mapped to the specific route.
Setting balance factor to 0 (default value) disables consideration of load situation.

### Required Changes

#### Gorouter
- The Gorouter MUST be extended to take a regular sample expression via request header
- The Gorouter MUST implement a new `EndpointIterator` to calculate hash, based on the provided expression 
- The Gorouter MUST consider consistent hashing 
- The Gorouter SHOULD locally cache the computed hash values to avoid expensive recalculations for each request for which 
hash-based routing should be applied
- The Gorouters SHOULD NOT implement a shared hash cache
- The Gorouter MUST assess the current request load across all application instances mapped to a particular route in order to prevent overload situations
- The Gorouter MUST have something like hash circle to handle instance overload situation. If the application instance associated with the calculated hash exceeds the balance factor of the average load across all instances, it is considered as overloaded.
Overflow traffic should be directed always to the same but not overloaded instance rather than to a random one, ensuring that high-load tenants do not revert to round-robin behavior.

#### Cloud Controller
The load balancing property of the [route object](https://v3-apidocs.cloudfoundry.org/version/3.190.0/index.html#the-route-options-object) 
MUST be extended to allow `hash:<header>:<balance-factor>` as well `hash:<header>` as valid values:

```bash
version: 1
applications:
- name: test
  routes:
  - route: test.example.com
    options:
      loadbalancing: hash:x-tenant-id:150
  - route: anothertest.example.com
    options:
      loadbalancing: hash:x-tenant-id
```

#### CF cli
- The `create-route`, `update-route`, `map-route` commands MUST be modified to allow the setting of the new load balancing algorithm.
```bash
cf create-route MY-APPexample.com --hostname test --option loadbalancing=hash:x-tenant-id:150
cf update-route MY-APP example.com --hostname test --option loadbalancing=hash:x-tenant-id:150
cf map-route MY-APP example.com --hostname test --option loadbalancing=hash:x-tenant-id:150
```
- The column options of the `routes`, `route` commands MUST be updated to display the new load balancing algorithm.

