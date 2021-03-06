# GRPC Components
[![Build Status](https://travis-ci.org/lstoll/grpce.svg?branch=master)](https://travis-ci.org/lstoll/grpce)

This repo has some experimental components for running services in a dynamic environment inside EC2. It's targeted at grpc-go, but would be easy to adapt to pretty much anything. It's intended to work reliably with litte infrastructure overhead, and without just having one magic shared cert/key everyone uses. The only external dependency is a KV store, which I intend to use S3. S3 is simple, reliable, and likely effective enough for this. Load balancing is handled by the client which eliminates the need for an ELB. Servers generate their own certificates and store the public component in the KV store, to avoid needing a CA infrastructure. This also still allows the revocation of per-server creds, and verifying that the server you connect to is the exact server. Client auth is handled by [Instance Identity Documents](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-identity-documents.html). This allows us to assert the account ID and instance ID of clients, without requiring any pre-shared state. This means instances can be managed inside an ASG, but not using a long-lasting shared key or other external service. All service discover is done via the KV store (i.e S3), with a model that is forgiving of it's consistency model

The end result should be a way to securely run services inside AWS without any long lasting or shared credentials, without requiring any infrastructure over other than a S3 bucket and some IAM/role policies.

## Components

### Polling KV Resolver

This is a resolver for the RoundRobin load balancer in GRPC. It
essentially takes a function that will return a list of addresses, and
an interval with which to call this. It will then provide the right
data to the balancer as hosts are added and removed.

```go
// This is the function we'll poll for the current list of servers
lookup := func(key string) ([]string, error) {
	if key == "testtarget" {
		return []string{"server1.abc.com", "server2.abc.com"}, nil
	}
	return nil, fmt.Errorf("Unknown target: %q", key)
}

conn, err := grpc.Dial("testtarget",
	grpc.WithBalancer(grpc.RoundRobin(kvresolver.New("testtarget", 10*time.Second, lookup))))
```

### Dynamic certs

This is a TLS credential implementation for the server that generates
a self-signed cert on the fly. This will get persisted in a KV
store. The client implementation looks up the cert based on the
address from the KV store, and validates it at the root cert for this
connection.

```go
// store matches a Get/Put/Delete interface.
s := grpc.NewServer(grpc.Creds(kvcertverify.NewServerTransportCredentials(store, address, time.Now().AddDate(0, 0, 1))))

// The client gets the same store.
conn, err := grpc.Dial(address, grpc.WithTransportCredentials(kvcertverify.NewClientTransportCredentials(store)))
```

### Handshake transport auth

This is intended to be a generic component that can be used to perform any kind of handshake over the connection when it's established, but before gRPC consumes it. It can wrap another transport (i.e the dynamic certs).

### Instance Identity Document Verification.

Utilities for verifying AWS Instances' [Instance Identity Documents](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-identity-documents.html). This provides a method to fetch the document and pkcs7 signature fromt the Instance Metadata server, which clients can use to retrive them. It also provides a method to check the document & signature against AWS's Cert, returning relevant fields

```go
// Verify them, returning the document
iddoc, err := VerifyDocumentAndSignature(doc, sig)
if err != nil {
	fmt.Println(iddoc.InstanceID)
}
```

### go-metrics Reporting Interceptors

Interceptors that will report stats about the server to a go-metrics registry

```
s := grpc.NewServer(
	grpc.StreamInterceptor(NewStreamServerInterceptor(registry, "p")),
	grpc.UnaryInterceptor(NewUnaryServerInterceptor(registry, "p")),
)
```

### h2c

Server and corresponding Dialer types for managing an h2c upgrade over a HTTP 1.1 endpoint.

```
s := grpc.NewServer()
helloproto.RegisterHelloServer(s, helloH2C{})

srv := &h2c.Server{
	HTTP2Handler:      s,
	NonUpgradeHandler: http.HandlerFunc(http.NotFound),
}

go http.Serve(ln, srv)
```

```
conn, err := grpc.Dial(ln.Addr().String(), grpc.WithDialer(h2c.Dialer{}.DialGRPC), grpc.WithInsecure())
```


## Flow design

This is the model I was targeting

![Diagram representing flow in system](https://cdn.lstoll.net/screen/grpcexperiments_flow.html_-_draw.io_2016-06-11_14-29-17.png)


## Security

The instance identity document security model and code hasn't really been audited or had deep thought put in to it. The concept works in my head, but I need an external opinion and to think about all the risks. The certificate storage is potentially easier to reason about, the model for server verification depends on managing access to write and update to the KV store.
