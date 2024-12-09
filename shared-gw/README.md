<H1>Shared Gateway pattern</H1>

This pattern allows a platform team to define application namespaces that can be delegated to individual application teams.

An application team may have one or more assigned namespaces. Application teams are assigned permissions only on their namespaces.

When the platform team creates namespaces they label the application team's namespaces to align with a particular application (in this case, foo and bar).

Each application has listeners defined in the GKE Gateway - with a unique subdomain. This avoids any possible route duplication between applications. A wildcard cert could be used to protect the Gateway in this case.

Via `allowedRoutes` the application teams can ONLY create `HTTPRoute` for the platform-managed gateway for their subdomain.

To test:

```curl -H "host: foo.chilm.com" http://[GATEWAY_IP_ADDRESS]/```

```curl -H "host: bar.chilm.com" http://[GATEWAY_IP_ADDRESS]/```
