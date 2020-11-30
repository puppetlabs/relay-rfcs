# Identity management platform (IdP) comparison

## Auth0

[Auth0](https://auth0.com) is a hosted IdP service used by quite a few big names.
This is a hosted service that we would need to make requests to from the UI and
API. These requests would leave our cluster and user data and permissions would
be stored with a 3rd party entity.

### Advantages

- No infrastructure or servers to manage.
- Supports user/pass, social OAuth logins, broad single sign-on support.
- Normalized user profile data that matches Relay user profiles for the most
  part.
- Custom login and registration styles.
- Supports remote login forms, but it's not recommended for security reasons.
  Therefore we should not expect to use the design system for user login and
  registration.
- Supports WebAuthn (Yubikeys and hardware security keys)
- Has a REST API for updating entities.
- Styling seems easy, but limited. It can be done through the UI, so small
  changes won't require a deploy.

### Disadvantages

- Additional latency for all authenticated requests.
- Harsh log retention policy.
- Per-seat licencing which can get quite expensive.
- RBAC permissions are tied to things called APIs. These are either endpoints or
  some unique identifier for a resource. This means we would need to restructure
  how we approach RBAC, but I think it would be a better design.
- Requires extra work to support the dev environment.
- We hand some of our user data to a 3rd party, which might not be desirable for
  some enterprise customers. This can be combatted with a proper interface that
  can use multiple IdP's, allowing a customer to request integration with
  another identity management service.

### Clients and libraries

- Go: None, but has an example repo: https://github.com/auth0-samples/auth0-golang-web-app
- Javascript: https://github.com/auth0/auth0.js/

## Keycloak

[Keycloak](https://www.keycloak.org/) is a self-hosted IdP used by some Puppet
teams and also a few big names. We would run this server in our own cluster and
make requests to it over the Kubernetes network. User data and permissions would
be stored in a separate Keycloak-managed database controlled by our
infrastructure management tools.

### Advantages

- Minimal additional latency on authenticated requests, co-located (or can be)
  with our server components that will utilize it.
- Full control over customer and user data.
- Full control over log data with no retention constraints.
- Supports user/pass, social OAuth logins, broad single sign-on support.
- Normalized user profile data that matches Relay user profiles for the most
  part.
- Supports WebAuthn (Yubikeys and hardware security keys)
- Custom login and registration styles.
- Has a REST API for updating entities.
- Since we would control the install, we would not need to do much work to get
  identity management working in a development environment.

### Disadvantages

- Additional infrastructure to manage.
- Adding additional data to users and profiles is a little clunky through their
  attribute system (glorified key/values on entity objects).
- Styling and templating looks like it might require a deploy because it's
  compiled into the Keycloak server. This isn't too big of a deal because it
  won't happen often, but something to consider.

### Clients and libraries

- Go: https://github.com/Nerzal/gocloak 
- Javascript: https://www.keycloak.org/docs/latest/securing_apps/#_javascript_adapter

## Conclusion

I'm leaning towards Auth0 because of the initial cost (as in time) savings for
implementation, BUT implementing this will need to include a development
environment adapter to make sure we can still use the API anywhere that isn't
staging or production. The RBAC story for Auth0 is weaker than Keycloak, but I
think it covers our needs in the Relay SaaS.

Like I mentioned in the disadvantages section above, there is a latency cost for
requests and we should also consider how we control egress from our edge API. We
might have some security policy constraints that will make integration with
Auth0 harder and if that is the case, we should consider integrating with
Keycloak instead.
