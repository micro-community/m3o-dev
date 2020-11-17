# Identity

Users and services in micro are both identified by holding a JWT that
corresponds to an [account](auth.md). Accounts for users are created
by either signing up with a standard email / password or going through
an Ouath2 flow with an external identity provider. Each service has an automatically generated implicit account.

All authentication / authorization features from the [account](auth.md)
design can be applied to services in the same way as users.

## Overview

The Auth service holds the cryptographic key material that can sign
account tokens. Identity providers validate a user or service is who
they say they are, then (using pre-generated static credentials) ask the
Auth service for an account / token, returning them to the requester.

To begin with, there will be 3 Identity providers

  * Oauth (GitHub / Google)
  * Basic (username+password)
  * Runtime (service accounts)
 
Every go-micro service is initialised with a name. On start-up, the
service name and node (a UUID for that instance of the service) is
validated, and an account is generated for the lifetime of said
instance. The account will have `type: service` and `provider: runtime`
set

## Design

Identity in micro is tied to an `Account` generated by the Auth service.
The `Account.Generate` endpoint may only be called by an Identity
provider that is already known to the Auth service (using offline / pre-generated accounts / secrets).

It is expected that as part of deploying `micro auth`, static accounts for all identity providers are created.

```go
// Provider is a micro identity provider
type Provider interface {
  // Generate validates the caller using options passed in via a URI
  // then asks the auth service to generate an account.
  Generate(uri string) (Token, error)
}
```

```protobuf
service Provider {
 	rpc Generate (GenerateRequest) returns (GenerateResponse) {}; 
}

message GenerateResponse {
	string state = 1;
}

message GenerateResponse {}
```

The micro runtime will implement the identity provider interface for
service accounts.

An example flow for service identity generation is as follows:

  1. The runtime service receives a request to run the service `foo`.
  2. Service is started, e.g. by creating a kubernetes deployment with
     an injected cert/key pair.
  3. As part of `service.Init()` the service calls the identity provider
     (runtime in this case) identifying itself with an x.509 certificate.
  4. The provider calls `auth.Generate()` on behalf of the service.
  5. (Optionally) when the service is deleted, the runtime
     `auth.Destroy()`s the account

Any errors in the process should be handled gracefully and returned to
the service, so that intelligent backoff can be used inside
`service.Init()`. For example, the auth Service being unavailable would
be a retryable error, bad callback state should cause the service to die.