# Cyrus

> PKI registration authority for Kubernetes.

## Motivation

Securing inter-cluster traffic via TLS is important for two main reasons:
1. Encryption of data in transit via symmetric key encryption.
2. Verification of server identity by clients and optional identity verification of client by servers.

While the importance of TLS is recognized, it can be difficult to effectively and efficiently manage a PKI infrastructure in such a dynamic environment at any scale. To mitigate this difficultly, numerous approaches are taken. Here are a few that I have seen:
* Terminate TLS at the edge of a cluster and forgo the use of TLS for inter-cluster traffic.
* Use a [Mutating Webhook](https://kubernetes.io/docs/admin/admission-controllers/#mutatingadmissionwebhook-beta-in-19) to dynamically inject a _sidecar_ container into each pod that is responsible for TLS termination for inter-cluster traffic. _Note that if you are already using a service mesh, namely [Istio](https://github.com/istio), then this functionality is already [part of that project](https://github.com/istio/istio/tree/master/security) and you should utilize that instead._
* Use the same certificate authority and certificate/key pair to secure all inter-cluster or inter-namespace traffic. Note that this approach is not a complete mitigation however as it omits the verification component of TLS.

The goal of this project is to function as a registration authority and provide dynamic TLS assets for inter-cluster traffic. This project aims to accomplish this without the need of a _sidecar_ container and without minimizing the benefits that TLS provides.

## How it Works

Cyrus works in the following manner. Assume we apply a new service as shown below. Because the annotation `cyrus.io/enabled` is `true`, a new set of TLS assets containing the correct `IP` and `DNS` subject alternative names (SANS) will be created. These assets will be persisted via a Kubernetes secret resource of type `kubernetes.io/tls`. This secret will also contain the CA certificate so that a client's identity can be verified if the use case permits.

Now that we have a new set of TLS assets created in the form of a secret, we must ensure that all pods routable through this service have this secret mounted. Cyrus handles this as one of two cases. If there are pods that already exist that are routable by this service, then Cyrus will update the existing pod with the additional volume mount and the pod will be restarted. For pods not yet created, a mutating webhook will be used to add this volume mount so that mutation occurs before startup. Note that the default mount location can be overwritten with the `cyrus.io/location` annotation.

If it is decided that this mutating behavior is too invasive, a non-mutating approach may be taken by setting the `cyrus.io/mutating` annotation to `false`. In this mode, Cyrus expects that all pods routable by this service properly mount the secret specified by the `cryrus.io/secret` annotation.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: foo
  annotations:
    cyrus.io/enabled: 'true'
    cyrus.io/location: '/etc/pki'
    cyrus.io/mutating: 'false'
    cryrus.io/secret: 'secretName'
spec:
  selector:
    app: foo
  ports:
  - port: 443
```

## Assumptions

Cyrus makes some assumptions which are listed below:

* It is assumed that inter-cluster traffic will occur by referencing either the service's DNS record(s) or cluster IP. It is assumed that a specific pod's DNS record(s) or IP are not used.
