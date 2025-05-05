---
title: "Using Pinniped for authentication against Talos-based Kubernetes clusters"
date: "2025-05-03"
description: "Usinig Pinniped to access Kubernetes on Talos; what it is, how it works and why I chose it"
showtoc: true
hideSummary: true
categories:
  - "homelab"
  - "kubernetes"
tags:
  - "homelab"
  - "kubernetes"
  - "authentication"
  - "kubectl"
  - "commandline"
---

## What we will learn

This post will cover the basics of the various Pinniped components; what they are responsible for and how they all work together. It will also detail how I configured and deployed Pinniped to authenticate access to the clusters in my homelab, as well as highlight an important gotcha when running Pinniped on Talos-based clusters.

## Why am I doing this?

Whilst I was reworking my homelab setup in order to migrate from Kubeadm clusters (which were manually provisioned) to declarative Cluster API-built clusters, I decided it was also time to overhaul how I access my clusters too. What's a little scope creep between clusters? My previous solution involved using [dex](https://github.com/dexidp/dex) and [dex-k8s-authenticator](https://github.com/mintel/dex-k8s-authenticator) to authenticate me via GitHub; this worked _OK_ however it got a little tedious having to manually access a webpage and then copy and paste commands to update the kubeconfig each time I needed to re-authenticate.

Whilst searching for a better authentication flow, I discovered [Pinniped](https://pinniped.dev/). Pinniped is an open-source authentication system designed to work with Kubernetes clusters, and it provides a unified way to authenticate users across multiple clusters using various identity providers.

Whilst I use Talos as my OS of choice, all of the information in this post will apply to pretty much any Kubernetes cluster regardless of distribution. However, there is a small gotcha I discovered when deploying Pinniped on clusters bootstrapped without the aid of [Kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/) which is detailed towards the end.

## Why Pinniped?

### Features

Firstly, let's go over what Pinniped offers:

- Unified authentication: Pinniped allows users to authenticate once with their chosen identity provider and then access multiple Kubernetes clusters without having to log into each one (for as long as the credentials are valid).

- Integration with identity providers: It supports integration with any identity provider which uses the OpenID Connect (OIDC) standard, as well as LDAP and Active Directory, making it easy to use existing enterprise identity systems for Kubernetes authentication.

- Smooth user experience: Integration with kubectl means users are interactively prompted to log in. Additionally, kubeconfig files do not contain static user credentials which improves security and ease of use.

- Short-lived credentials: All credentials issued are short-lived and frequently refreshed, ensuring that access can be quickly revoked if a user loses authorisation in the identity provider.

- Cluster-agnostic: Pinniped works with any Kubernetes distribution, whether running on-premises or in the cloud.

- Declarative configuration: Cluster admins can manage authentication policies using Kubernetes Custom Resource Definitions (CRDs), enabling GitOps workflows and easy reconfiguration.

### Why I chose Pinniped

Using a heavyweight system like Pinniped for a homelab with just a handful of clusters may seem like overkill, but I have a habit of engineering my homelab to the highest (reasonable) standard. It's also a learning experience - what's a homelab for if it's not to experiment with new technologies and tools?

Besides, a colleague of mine once said that my home setup is more elaborate and fully-featured than some customer setups, so I have a lot to live up to.

## Pinniped architecture

Pinniped consists of two main components; the Supervisor and the Concierge.

### Supervisor

The Supervisor acts as an OIDC server which is responsible for authenticating users with the configured identity provider and then issuing federated ID tokens which are presented to the target cluster's Concierge.

The Supervisor is the central hub which is responsible for authenticating users and as such is the most important Pinniped component from a security standpoint. Because of this, it is intended to be deployed in a cluster which isn't accessible to less-privileged users. For the purposes of my homelab, I run the Supervisor in my Cluster API (hereby referred to as CAPI) [Management Cluster](https://cluster-api.sigs.k8s.io/user/concepts#management-cluster) - as I'm the only person interacting with these clusters, separation of security concerns is somewhat of a moot point.

### Concierge

If the Supervisor is the hub, then the Concierge is the spoke. This runs in every cluster which users need to access and is responsible for credential exchange. Once the user has authenticated with the Supervisor, a token is passed to the Concierge which then converts it into a credential which Kubernetes understands. In my case, this means that my Management Cluster runs both the Supervisor and the Concierge, but any other cluster only needs to run the Concierge.

### Supporting resources

#### Identity Provider

The Identity Provider resource tells the Supervisor where and how to authenticate users. As I'm using GitHub for authentication, I have configured a `GitHubIdentityProvider` ([which you can see here](https://github.com/a7d-corp/homelab-clusters-fleet/blob/main/flux/apps/base/pinniped-supervisor/post-setup/github-identity-provider.yaml})). On the GitHub side, [an App](https://pinniped.dev/docs/howto/supervisor/configure-supervisor-with-github/) is configured with the Supervisor's details.

#### Federation Domain

The `FederationDomain` is responsible for configuring the Supervisor as an OIDC provider by associating it with the Identity Provider (or Providers, if you configure more than one). As well as linking the Identity Provider(s) together, it also details how the Supervisor exposes the Domain to clients. Configuration for my setup can be seen [here](https://github.com/a7d-corp/homelab-clusters-fleet/blob/main/flux/apps/base/pinniped-supervisor/post-setup/federation-domain.yaml).

#### JWT Authenticator

In order for the Concierge to validate the tokens issued by the Supervisor (and to ensure that they are intended for use with this cluster), a `JWTAuthenticator` is required in each target cluster.

The JWT Authenticator has several ways of verifying the user's token:

- Issuer matching: The JWTAuthenticator is configured with the exact issuer URL of the Supervisor's FederationDomain. When a token is presented, the Concierge checks that the `iss` (issuer) claim in the JWT matches the configured issuer value.

- Audience verification: Each cluster's JWTAuthenticator is configured with a unique audience value. The JWT's `aud` (audience) claim must match this value to ensure the token was intended for that specific cluster. This prevents tokens from being reused across clusters unintentionally.

- Signature validation: The JWTAuthenticator verifies the token's signature using the public key or certificate authority (CA) information associated with the Supervisor.

- Expiration and validity checks: The Concierge checks the token's `exp` (expiration) field and other standard JWT claims to ensure it is still valid.

## Interaction between kubectl and the Concierge

Whilst the function of the Supervisor is _fairly_ simple, how the Concierge works is a little more complex, depending on how you configure your cluster. Which of the following options you choose depends on whether you are comfortable allowing unauthenticated access to the API server.

### With API server anonymous authentication enabled

It is possible to enable anonymous authentication to the Kubernetes API server by setting the `--anonymous-auth=true` flag on the API server, however as the flag implies, this allows certain unauthenticated requests to the API to succeed.

With anonymous authentication enabled, the Concierge will use the Token Credential Request API. This method will only work for Clusters where the control plane nodes are part of the cluster as it involves a custom pod running alongside the `kube-controller-manager` - this isn't possible on most managed Kubernetes clusters.

### Without anonymous authentication enabled

Without anonymous authentication enabled, the Concierge runs in Impersonation Proxy mode, which will work on any cluster.

Impersonation Proxy functionality:

- The Impersonation Proxy presents an HTTPS endpoint that receives API requests from clients (such as kubectl or the Pinniped CLI).

- After authenticating the user's token (issued by the Supervisor), the Proxy forwards the request to the Kubernetes API server.

- To achieve this, the Concierge uses Kubernetes’ built-in impersonation features. It sets HTTP headers (Impersonate-User, Impersonate-Group) to make the API server treat the request as if it originated from the authenticated user, rather than the Proxy.

As the Proxy sits in front of the API server, it must be exposed to the client via a LoadBalancer or an Ingress in order to receive the requests. The Concierge can automatically provision the LoadBalancer (which is most useful in cloud environments), or alternatively you can disable this functionality and route requests to the Proxy yourself via any other method.

## Deployment

I chose to use the [Bitnami Helm chart](https://artifacthub.io/packages/helm/bitnami/pinniped) as my method of deployment, so the following section will relate to this (however the implementation details are the same regardless of the deployment method chosen).

Note that the Bitnami chart is used to deploy both the Supervisor _and_ the Concierge components.

### Supervisor deployment

The Supervisor's deployment is [quite straightforward](https://github.com/a7d-corp/homelab-clusters-fleet/blob/main/flux/apps/base/pinniped-supervisor/helmrelease.yaml). A couple of things to note:

- the Ingress URL is the endpoint configured as the [Issuer address](https://github.com/a7d-corp/homelab-clusters-fleet/blob/main/flux/apps/base/pinniped-supervisor/post-setup/federation-domain.yaml#L7) in the FederationDomain.
- I wanted to deploy the Concierge in a different namespace so I [disabled Concierge deployment here](https://github.com/a7d-corp/homelab-clusters-fleet/blob/main/flux/apps/base/pinniped-supervisor/helmrelease.yaml#L21).

### Concierge deployment

Configuration of the Concierge is [slightly more complex](https://github.com/a7d-corp/homelab-clusters-fleet/blob/main/flux/apps/base/pinniped-concierge/helmrelease.yaml), mostly because Bitnami's chart doesn't actually have any configuration options for the CredentialIssuer - instead you have to pass them in as a map of values which are then templated directly into the CredentialIssuer resource spec:

```yaml
credentialIssuerConfig:
  impersonationProxy:
    externalEndpoint: concierge.domain.com
    mode: enabled
    service:
      type: ClusterIP
```

- `mode`: leaving this set to `auto` will enable the Impersonation Proxy depending on whether the cluster configuration requires it (as discussed earlier with regards to anonymous authentication). I chose to explicitly enable it as I do not enable anonymous auth to my clusters.
- `externalEndpoint`: this should be set to the URL used to expose the Concierge to clients when the Concierge isn't configured to automatically provision a LoadBalancer.
- `service.type`: setting this to ClusterIP causes a Service to be created which targets the Concierge (rather than a LoadBalancer).

I chose to disable automatic LoadBalancer creation because I wanted to expose the Concierge via my existing Ingress controller deployment as this simplifies DNS management in my homelab. Because of this, I also need to manually create an [Ingress](https://github.com/a7d-corp/homelab-clusters-fleet/blob/main/flux/apps/base/pinniped-concierge/ingress-impersonation-proxy.yaml) which targets the Impersonation Proxy's Service.

## Client configuration

In order to use Pinniped, clients must install the [Pinniped CLI](https://pinniped.dev/docs/howto/install-cli/). This is used for a variety of tasks, but the most important point to note is that it is a kubectl [credential plugin](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#client-go-credential-plugins) and will be referenced in the kubeconfig.

### Obtaining the client kubeconfig

To generate the client kubeconfig, you must first ensure that you are using an admin kubeconfig for the target cluster. As the admin, run the following command which will output the kubeconfig to stdout:

```bash
pinniped get kubeconfig --concierge-mode ImpersonationProxy
```

If you configured the Concierge to use the TokenCredentialRequest method then you can omit the `--concierge-mode` flag (as this is the default).

Some interesting things to note in the kubeconfig:

- the cluster's `server` field will contain the URL of the Concierge ingress configured earlier, rather than the cluster's API server address (assuming you are using the Impersonation Proxy).
- the cluster's CA field contains the CA created by the Concierge. Checking the certificate's Issuer field shows the following: `issuer=CN=Pinniped Impersonation Proxy Serving CA`.
- the `user` is configured to use the Pinniped CLI binary and as such it passes a bunch of arguments in order to tell the CLI where the Supervisor and Concierge are running and how they're configured.

### A gotcha when deploying to Talos clusters

Whilst testing Pinniped out I hit a snag when trying to retrieve the kubeconfig which required a little troubleshooting to solve. The root cause is due to the fact that Talos doesn't use Kubeadm to bootstrap Kubernetes, so the following will apply to any Kubernetes distribution which uses a bootstrap provider other than Kubeadm.

During bootstrap, Kubeadm shares some information between nodes using the [`kube-public` namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/#initial-namespaces), specifically via the `cluster-info` ConfigMap. In my legacy clusters which _were_ created by Kubeadm, this just contained scaffolding for a kubeconfig so I hand-crafted it myself and [deployed it via Flux](https://github.com/a7d-corp/homelab-clusters-fleet/blob/main/flux/clusters/room101-a7d-mc/cluster-prereqs/configmap-cluster-values.yaml) - this satisfied Pinniped and allowed me to retrieve the kubeconfig.

## Testing it all out

With this all in place, it's time to actually use Pinniped to access the cluster. Using kubectl as normal will cause the Pinniped CLI to be invoked under the hood. This checks if we already have an existing session, and if not a web browser is automatically opened which directs us to authenticate with the configured Identity Provider. Once the authentication flow is complete, our kubectl command succeeds:

```
➜  kubectl get no
Log in by visiting this link:

    https://supervisor.room101-a7d-mc.lab.a7d.dev/room101/oauth2/authorize?access_type=offline...cb037e76827bd8b51739c

    Optionally, paste your authorization code: [...]

NAME                                 STATUS   ROLES           AGE     VERSION
room101-a7d-mc-cp-e2cc5a-5lws8       Ready    control-plane   7d12h   v1.32.3
room101-a7d-mc-cp-e2cc5a-7vwf9       Ready    control-plane   6d12h   v1.32.3
room101-a7d-mc-cp-e2cc5a-wnppn       Ready    control-plane   7d12h   v1.32.3
room101-a7d-mc-workers-rf854-bpm6x   Ready    <none>          4d23h   v1.32.3
room101-a7d-mc-workers-rf854-d4882   Ready    <none>          8d      v1.32.3
room101-a7d-mc-workers-rf854-p97rl   Ready    <none>          8d      v1.32.3
```

### Authorisation

It's important to note that this post has only covered authentication (identity verification); authorisation (what we are allowed to do once authenticated) is handled by Kubernetes [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/). In my case I am the sole user of the clusters and so I [grant `cluster-admin` access](https://github.com/a7d-corp/homelab-clusters-fleet/blob/main/flux/configs/room101-a7d-mc/rbac/pinniped-a7d-corp-admins-clusterrolebinding.yaml) to anyone (i.e. me) who is in the `admins` team in my GitHub organisation.