# General RPC conventions

**Status**: [Experimental](../../document-status.md)

The conventions described in this section are RPC specific. When RPC operations
occur, metric events about those operations will be generated and reported to
provide insight into those operations. By adding RPC properties as attributes
on metric events it allows for finely tuned filtering.

<!-- Re-generate TOC with `markdown-toc --no-first-h1 -i` -->

<!-- toc -->

- [Metric instruments](#metric-instruments)
  * [RPC Server](#rpc-server)
  * [RPC Client](#rpc-client)
- [Attributes](#attributes)
  * [Service name](#service-name)
- [gRPC conventions](#grpc-conventions)

<!-- tocstop -->

## Metric instruments

The following metric instruments MUST be used to describe RPC operations. They
MUST be of the specified type and units.

*Note: RPC server and client metrics are split to allow correlation across client/server boundaries, e.g. Lining up an RPC method latency to determine if the server is responsible for latency the client is seeing.*

### RPC Server

Below is a table of RPC server metric instruments.

| Name | Instrument | Unit | Unit ([UCUM](README.md#instrument-units)) | Description | Status | Streaming |
|------|------------|------|-------------------------------------------|-------------|--------|-----------|
| `rpc.server.duration` | Histogram  | milliseconds | `ms` | measures duration of inbound RPC | Recommended | N/A.  While streaming RPCs may record this metric as start-of-batch to end-of-batch, it's hard to interpret in practice. |
| `rpc.server.request.size` | Histogram  | Bytes | `By` | measures size of RPC request messages (uncompressed) | Optional | Recorded per message in a streaming batch |
| `rpc.server.response.size` | Histogram  | Bytes | `By` | measures size of RPC response messages (uncompressed) | Optional | Recorded per response in a streaming batch |
| `rpc.server.requests_per_rpc` | Histogram  | count | `{count}` | measures the number of messages received per RPC.  Should be 1 for all non-streaming RPCs | Optional | Required |
| `rpc.server.responses_per_rpc` | Histogram  | count | `{count}` | measures the number of messages sent per RPC.  Should be 1 for all non-streaming RPCs | Optional | Required |

### RPC Client

Below is a table of RPC client metric instruments.  These apply to traditional
RPC usage, not streaming RPCs.

| Name | Instrument | Unit | Unit ([UCUM](README.md#instrument-units)) | Description | Status | Streaming |
|------|------------|------|-------------------------------------------|-------------|--------|-----------|
| `rpc.client.duration` | Histogram | milliseconds | `ms` | measures duration of outbound RPC | Recommended | N/A.  While streaming RPCs may record this metric as start-of-batch to end-of-batch, it's hard to interpret in practice. |
| `rpc.client.request.size` | Histogram | Bytes | `By` | measures size of RPC request messages (uncompressed) | Optional | Recorded per message in a streaming batch |
| `rpc.client.response.size` | Histogram | Bytes | `By` | measures size of RPC response messages (uncompressed) | Optional | Recorded per message in a streaming batch |
| `rpc.client.requests_per_rpc` | Histogram | count | `{count}` | measures the number of messages received per RPC.  Should be 1 for all non-streaming RPCs | Optional | Required |
| `rpc.client.responses_per_rpc` | Histogram | count | `{count}` | measures the number of messages sent per RPC.  Should be 1 for all non-streaming RPCs | Optional | Required |

## Attributes

Below is a table of attributes that SHOULD be included on metric events and whether
or not they should be on the server, client or both.

<!-- semconv rpc -->
| Attribute  | Type | Description  | Examples  | Required |
|---|---|---|---|---|
| [`rpc.system`](../../trace/semantic_conventions/rpc.md) | string | A string identifying the remoting system. See below for a list of well-known identifiers. | `grpc` | Yes |
| [`rpc.service`](../../trace/semantic_conventions/rpc.md) | string | The full (logical) name of the service being called, including its package name, if applicable. [1] | `myservice.EchoService` | No, but recommended |
| [`rpc.method`](../../trace/semantic_conventions/rpc.md) | string | The name of the (logical) method being called, must be equal to the $method part in the span name. [2] | `exampleMethod` | No, but recommended |
| [`net.peer.ip`](../../trace/semantic_conventions/span-general.md) | string | Remote address of the peer (dotted decimal for IPv4 or [RFC5952](https://tools.ietf.org/html/rfc5952) for IPv6) | `127.0.0.1` | See below |
| [`net.peer.name`](../../trace/semantic_conventions/span-general.md) | string | Remote hostname or similar, see note below. | `example.com` | See below |
| [`net.peer.port`](../../trace/semantic_conventions/span-general.md) | int | Remote port number. | `80`; `8080`; `443` | See below |
| [`net.transport`](../../trace/semantic_conventions/span-general.md) | string | Transport protocol used. See note below. | `ip_tcp` | See below |

**[1]:** This is the logical name of the service from the RPC interface perspective, which can be different from the name of any implementing class. The `code.namespace` attribute may be used to store the latter (despite the attribute name, it may include a class name; e.g., class with method actually executing the call on the server side, RPC client stub class on the client side).

**[2]:** This is the logical name of the method from the RPC interface perspective, which can be different from the name of any implementing method/function. The `code.function` attribute may be used to store the latter (e.g., method actually executing the call on the server side, RPC client stub method on the client side).

**Additional attribute requirements:** At least one of the following sets of attributes is required:

* [`net.peer.ip`](../../trace/semantic_conventions/span-general.md)
* [`net.peer.name`](../../trace/semantic_conventions/span-general.md)

`rpc.system` has the following list of well-known values. If one of them applies, then the respective value MUST be used, otherwise a custom value MAY be used.

| Value  | Description |
|---|---|
| `grpc` | gRPC |
| `java_rmi` | Java RMI |
| `dotnet_wcf` | .NET WCF |
<!-- endsemconv -->

To avoid high cardinality, implementations should prefer the most stable of `net.peer.name` or
`net.peer.ip`, depending on expected deployment profile.  For many cloud applications, this is likely
`net.peer.name` as names can be recycled even across re-instantiation of a server with a different `ip`.

For client-side metrics `net.peer.port` is required if the connection is IP-based and the port is available (it describes the server port they are connecting to).
For server-side spans `net.peer.port` is optional (it describes the port the client is connecting from).
Furthermore, setting [net.transport][] is required for non-IP connection like named pipe bindings.

[net.transport]: ../../trace/semantic_conventions/span-general.md#nettransport-attribute

### Service name

On the server process receiving and handling the remote procedure call, the service name provided in `rpc.service` does not necessarily have to match the [`service.name`][] resource attribute.
One process can expose multiple RPC endpoints and thus have multiple RPC service names. From a deployment perspective, as expressed by the `service.*` resource attributes, it will be treated as one deployed service with one `service.name`.

[`service.name`]: ../../resource/semantic_conventions/README.md#service

## gRPC conventions

For remote procedure calls via [gRPC][], additional conventions are described in this section.

`rpc.system` MUST be set to `"grpc"`.

[gRPC]: https://grpc.io/
