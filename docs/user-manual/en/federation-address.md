# Address Federation

## Introduction 

Address federation is like full multicast over the connected brokers, in that every message sent to address 
on `Broker-A` will be delivered to every queue on that broker, but like wise will be delivered to `Broker-B` 
and all attached queues there.


Address federation dynamically links to other addresses in upstream brokers. It automatically creates a queue on the remote address for itself, 
to which then it consumes, copying to the local address, as though they were published directly to it. 

The upstream brokers do not need to be reconfigured or the address, simply permissions to the address need to be 
given to the address for the downstream broker.


![Address Federation](images/federation-address.png)
Figure 1. Address Federation


## Topology Patterns


### Symmetric

![Address Federation](images/federation-address-symetric.png)
Figure 2. Address Federation - Symmetric

As seen above, a publisher and consumer are connected to each broker. 
Queues and thus consumers on those queues, can receive messages published by either publisher.

It is important in this setup to set `max-hops=1` to so that messages are copied only one and avoid cyclic replication.
If `max-hops` is not configured correctly, consumers will get multiple copies of the same message.

### Full Mesh

![Address Federation](images/federation-address-complete-graph.png)
Figure 3. Address Federation - Full Mesh

If not already spotted, the setup is identical to symemtric but simply where all brokers are symmetrically federating each other, creating a full mesh.

As illustrated, a publisher and consumer are connected to each broker. 
Queues and thus consumers on those queues, can receive messages published by either publisher.

As with symmetric setup, it is important in this setup to set `max-hops=1` to so that messages are copied only one and avoid cyclic replication.
If `max-hops` is not configured correctly, consumers will get multiple copies of the same message.


### Ring

![Address Federation](images/federation-address-ring.png)
Figure 4. Address Federation - Symmetric

In a ring of brokers each federated address is `upstream` to just one other in the ring. To avoid the cyclic issue, 
it is important to set `max-hops` to `n - 1` where `n` is the number of nodes in the ring. 
e.g. in the example above property is set to 5 so that every address in the ring sees the message exactly once.

Whilst this setup is cheap in regards to connections, it is brittle, in that if a single broker fails, the ring fails.

### Fan out

![Address Federation](images/federation-address-fan-out.png)
Figure 5. Address Federation - Fan Out

One master address (it would required no configuration) is linked to by a tree of downstream federated addresses, 
the tree can extend to any depth, and can be extended to without needing to re-configure existing brokers. 

In this case messages published to the master address can be received by any consumer connected to any broker in the tree.


## Configuring Address Federation

Federation is configured in `broker.xml`.

Sample Address Federation setup:


```
<federations>
    <federation name="eu-north-1" user="federation_username" password="32a10275cf4ab4e9">
        <upstream name="eu-east-1">
           <static-connectors>
              <connector-ref>eu-east-connector1</connector-ref>
              <connector-ref>eu-east-connector1</connector-ref>
           </static-connectors>
           <policy ref="news-address-federation"/>
        </upstream>
        <upstream name="eu-west-1" >
           <static-connectors>
              <connector-ref>eu-west-connector1</connector-ref>
              <connector-ref>eu-west-connector1</connector-ref>
           </static-connectors>
           <policy ref="news-address-federation"/>
        </upstream>
        
        <address-policy name="news-address-federation" max-hops="1" auto-delete="true" auto-delete-delay="300000" auto-delete-message-count="-1" transformer-ref="federation-transformer-3">
           <include address-match="queue.bbc.new" />
           <include address-match="queue.usatoday" />
           <include address-match="queue.news.#" />
        
           <exclude address-match="queue.news.sport.#" />
        </address-policy>
        
        <transformer name="news-transformer">
           <class-name>org.foo.NewsTransformer</class-name>
           <property key="key1" value="value1"/>
           <property key="key2" value="value2"/>
        </transformer>
    </federation>
</federations>

```

In the above setup downstream broker `eu-north-1` is configured to connect to two upstream brokers `eu-east-1` and `eu-east-2`, the credentials used for both connections to both brokers in this sample are shared, you can set user and password at the upstream level should they be different per upstream.

Both upstreams are configured with the same address-policy `news-address-federation`, that is selecting addresses which match any of the include criteria, but will exclude anything that starts `queue.news.sport`.

**It is important that federation name is globally unique.**

Let's take a look at all the `address-policy` parameters in turn, in order of priority.

- `name` attribute. All address-policies must have a unique name in the server.

- `include` the address-match pattern to whitelist addresses, multiple of these can be set. If none are set all addresses are matched.

- `exclude` the address-match pattern to blacklist addresses, multiple of these can be set.

- `max-hops`. The number of hops that a message can have made for it to be federated, see [Topology Patterns](#topology-patterns) above for more details.

- `auto-delete`. For address federation, the downstream dynamically creates a durable queue on the upstream address. This is used to mark if the upstream queue should be deleted once downstream disconnects, 
and the delay and message count params have been met. This is useful if you want to automate the clean up, though you may wish to disable this if you want messages to queued for the downstream when disconnect no matter what.

- `auto-delete-delay`. The amount of time in milliseconds after the downstream broker has disconnected before the upstream queue can be eligable for `auto-delete`.

- `auto-delete-message-count`. The amount number messages in the upstream queue that the message count must be equal or below before the downstream broker has disconnected before the upstream queue can be eligable for `auto-delete`.

- `transformer-ref`. The ref name for a transformer (see transformer config) that you may wish to configure to transform the message on federation transfer.

**note** `address-policy`'s and `queue-policy`'s are able to be defined in the same federation, and be linked to the same upstream.


Now look at all the `transformer` parameters in turn, in order of priority:

- `name` attribute. This must be a unique name in the server, and is used to ref the transformer in `address-policy` and `queue-policy`

- `transformer-class-name`. An optional transformer-class-name can be
  specified. This is the name of a user-defined class which implements the
  `org.apache.activemq.artemis.core.server.transformer.Transformer` interface.

  If this is specified then the transformer's `transform()` method will be
  invoked with the message before it is transferred. This gives you the opportunity
  to transform the message's header or body before it is federated.
  
- `property` holds key, value pairs that can be used to configure the transformer.



Finally look at `upstream`, this is what defines the upstream broker connection and the policies to use against it.

- `name` attribute. This must be a unique name in the server, and is used to ref the transformer in `address-policy` and `queue-policy`

- `user`. This optional attribute determines the user name to use when creating
  the upstream connection to the remote server. If it is not specified the shared
  federation user and password will be used if set.

- `password`. This optional attribute determines the password to use when
  creating the upstream connection to the remote server. If it is not specified the shared
  federation user and password will be used if set.

- `static-connectors` or `discovery-group-ref`. Pick either of these options to
  connect the bridge to the target server.

  The `static-connectors` is a list of `connector-ref` elements pointing to
  `connector` elements defined elsewhere. A *connector* encapsulates knowledge of
  what transport to use (TCP, SSL, HTTP etc) as well as the server connection
  parameters (host, port etc). For more information about what connectors are and
  how to configure them, please see [Configuring the
  Transport](configuring-transports.md).

  The `discovery-group-ref` element has one attribute - `discovery-group-name`.
  This attribute points to a `discovery-group` defined elsewhere. For more
  information about what discovery-groups are and how to configure them, please
  see [Discovery Groups](clusters.md).


- `ha`. This optional parameter determines whether or not this bridge should
  support high availability. True means it will connect to any available server
  in a cluster and support failover. The default value is `false`.
  
- `circuit-breaker-timeout` in milliseconds, When a connection issue occurs, 
as the single connection is shared by many federated queue and address consumers, 
to avoid each one trying to reconnect and possibly causing a thrundering heard issue, 
the first one will try, if unsuccessful the circuit breaker will open, 
returning the same exception to all, this is the timeout until the circuit can be closed and connection retried.

