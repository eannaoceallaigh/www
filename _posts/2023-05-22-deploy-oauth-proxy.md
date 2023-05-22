---
title: "Deploying OAuth2 Proxy with the Azure Active Directory provider and Traefik reverse proxy"
date: 2023-05-22T20:30:00+01:00
categories:
  - blog
tags:
  - oauth2-proxy
  - traefik
  - kubernetes
  - helm
  - azure
classes: wide
---

In this tutorial, we will be deploying OAuth2 Proxy to a kubernetes cluster using Azure Active Directory as the authentication provider and Traefik as the reverse proxy.

We will also be using FluxCD to deploy Kubernetes resources via Helm.

Requirements:

- domain name
- git repository [configured with flux](https://fluxcd.io/flux/get-started/)
- kubernetes cluster
- kubectl tool installed your computer

### What is Oauth2 Proxy?

> A reverse proxy and static file server that provides authentication using Providers (Google, GitHub, and others) to validate accounts by email, domain or group.

If you have an application you want to make available on the internet but you want to grant access only to authorised users, you can use OAuth2 Proxy to force visitors to authenticate with an authentication provider like Google or Microsoft.

The tool is mainly focused on using NGINX reverse proxy so much of the official documentation is around that technology. This guide will give you guidance on how to use OAuth2 Proxy with Traefik reverse proxy which requires some specific setup to work.

### Deploying OAuth2 Proxy to Kubernetes using Helm and Flux

There is an official helm chart provided by the developers of OAuth2 Proxy.

The source code for this can be found on [GitHub](https://github.com/oauth2-proxy/manifests).

The following kubernetes manifests will get you up and running but there are slight improvements to be made to these configs which we will explore later on in the guide e.g. mounting arguments using kubernetes secrets.

In your git repository, add the helm repository to install the chart on the cluster:

```
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: oauth2-proxy-charts
  namespace: default
spec:
  interval: 1h
  url: https://oauth2-proxy.github.io/manifests
  timeout: 3m
```

Next, create a helm release:
```
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: oauth2-proxy
  namespace: default
spec:
  releaseName: oauth2-proxy
  chart:
    spec:
      chart: oauth2-proxy
      version: 6.12.0
      sourceRef:
        kind: HelmRepository
        name: oauth2-proxy-charts
        namespace: default
  interval: 5m
  values:
    extraArgs:
      cookie-domain: ".domain.com, .domain.net"
      email-domain: "domain.com"
      pass-authorization-header: true
      pass-basic-auth: false
      pass-host-header: true
      pass-user-headers: true
      provider: "azure"
      redirect-url: https://oauth2-proxy-01.domain.com/oauth2/callback
      reverse-proxy: true
      set-authorization-header: true
      set-xauthrequest: true
      show-debug-on-error: true
      skip-provider-button: true
      silence-ping-logging: true
      upstream: static://202
      whitelist-domain: ".domain.com"
      client-id: "<CLIENT-ID>"
      client-secret: "<CLIENT-SECRET>"
      cookie-secret: <COOKIE-SECRET>"
      oidc-issuer-url: "https://sts.windows.net/<TENANT-ID>/"
      extra-jwt-issuers: "https://login.microsoftonline.com/<TENANT-ID>/v2.0=<CLIENT-ID>"
      azure-tenant: "<TENANT-ID>"
    redis:
      enabled: false
    ingress:
      annotations:
        traefik.ingress.kubernetes.io/router.tls: "true"
      className: traefik
      enabled: true
      hosts:
        - oauth2-proxy-01.domain.com
```

You can find explanations for what these arguments mean in the [OAuth2 Proxy docs](https://oauth2-proxy.github.io/oauth2-proxy/docs/configuration/overview) but I want to give you some pointers on why these particular arguments are useful / required.

| Argument| Example value | Why is it needed? |
| --- | --- | --- |
| azure-tenant | "abcd1234-5678-91ef-23gh-ij4kl5mn6o" | Your Azure AD tenant id |
| client-id | "abcd1234-5678-91ef-23gh-ij4kl5mn6o" | The client id of the app registration you will have created in Azure AD |
| client-secret | "abcdefghijklmnopqrstuvwxyz1234456789?~!@$%" | The client secret you will have generated for the app registration in Azure AD |
| cookie-domain | ".domain.com, .domain.net" | So you cache cookies for the correct domain. It can be set to "*" to allow all domains |
| cookie-secret | "wj7fY0MwSPKZZoxUaUNnoeqgN0lEr5f-F6vGIFO1AR4=" | A randomly generated secret from the command line on your computer. See [here](https://oauth2-proxy.github.io/oauth2-proxy/docs/configuration/overview#generating-a-cookie-secret) |
| email-domain | "domain.com" | Can be used to restrict sign in to only certain domains. Use "*" to allow all domains |
| extra-jwt-issuers | "https://login.microsoftonline.com/YOUR-TENANT-ID/v2.0=YOUR-CLIENT-ID" | A list of extra JSON Web Token (JWT) issuer URLs. This is needed as there are two endpoints that Azure AD offers for authentication  |
| oidc-issuer-url | "https://sts.windows.net/YOUR-TENANT-ID/" | The OpenID Connect issuer URL for your provider, in this case, Azure AD |
| pass-authorization-header | true | Pass OIDC IDToken to upstream via Authorization Bearer header |
| pass-basic-auth | false | Pass HTTP Basic Auth, X-Forwarded-User, X-Forwarded-Email and X-Forwarded-Preferred-Username information to upstream. Set to `false` for Azure AD. |
| pass-host-header | true | Pass the request Host Header to upstream |
| pass-user-headers | true | Pass X-Forwarded-User, X-Forwarded-Groups, X-Forwarded-Email and X-Forwarded-Preferred-Username information to upstream |
| provider | "azure" | The authentication provider you want to use e.g. `google` for Google or `azure` for Azure AD |
| redirect-url | https://oauth2-proxy-01.domain.com/oauth2/callback | The redirect URL for OAuth2 Proxy. Tells your request where to go once authenticated |
| reverse-proxy | true | Tells OAuth2 Proxy that it's running behind a reverse proxy i.e. Traefik |
| set-authorization-header | true | Set Authorization Bearer response header (useful in Nginx auth_request mode) |
| set-xauthrequest | true | Set X-Auth-Request-User, X-Auth-Request-Groups, X-Auth-Request-Email and X-Auth-Request-Preferred-Username response headers (useful in Nginx auth_request mode). When used with --pass-access-token, X-Auth-Request-Access-Token is added to response headers. |
| show-debug-on-error | true | Show more informative error messages in the kubernetes pod logs |
| silence-ping-logging | true | Hides ping messages in the log. These aren't very useful and clutter the log |
| skip-provider-button | true | There is an option to show an interstitial page giving you the option to choose your authentication provider. Useful if you are using multiple authentication methods |
| upstream | static://202 | The upstream page to show when you've connected and authenticated directly to the OAuth2 Proxy. You could set this to an application URL if you want to redirect to a default application but this static page is enough to tell you the OAuth2 Proxy is working. When you authenticate, it will show a simple page saying `Authenticated` |
| whitelist-domain | ".domain.com" | Domain names you want to allow redirection to. Similar to email domains but for the destination application rather than the user |

Then, create the required middlewares:
```
---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: oauth-auth
  namespace: default
spec:
  forwardAuth:
    address: http://oauth2-proxy.default/oauth2/auth_or_start
    trustForwardHeader: true
    authResponseHeaders:
      - X-Auth-Request-Access-Token
      - Authorization
      - X-Auth-Request-User
      - X-Auth-Request-Email
      - X-Auth-Email
      - X-Auth-Username
---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: oauth-headers
spec:
  headers:
    sslRedirect: true
    stsSeconds: 315360000
    browserXssFilter: true
    contentTypeNosniff: true
    forceSTSHeader: true
    stsIncludeSubdomains: true
    stsPreload: true
    frameDeny: true
```

The middlewares are used to intercept the request and send it to the OAuth2 Proxy when it detects that you are not authenticated by means of a `401 Unauthorized` http status code being returned.

The forwardAuth address can be set to a fully qualified domain name but it should be noted that DNS for that URL will be required. In the example above, we are using the cluster DNS name of the kubernetes service which will always work. If you use a FQDN and the DNS changes or is unreachable, the application won't work.

The cluster DNS name for the service follows the format `<servicename>-<namespace>`.

The official docs have an [example](https://oauth2-proxy.github.io/oauth2-proxy/docs/configuration/overview#forwardauth-with-401-errors-middleware) that uses the `/oauth2/auth` endpoint in the address, however, I have found that this does not always work, so you should use the `/oauth2/auth_or_start` endpoint instead as per my example.

See also: [https://github.com/oauth2-proxy/oauth2-proxy/issues/46](https://github.com/oauth2-proxy/oauth2-proxy/issues/46)