# Custom user provider

## User entity
Use the maker bundle to create a new User class that implements UserInterface. See : https://symfony.com/doc/current/security.html#a-create-your-user-class

## Configure config/packages/security.yml
```yaml
providers:
  users:
      entity:
          class: App\Entity\User # The class used for our provider
          property: 'email' # The property used for login (default is username)
          
firewalls:
  api:
      pattern:   ^/api
      stateless: true
      provider: users  # Here we tell our firewall to use our new provider 
      guard:
          authenticators:
              - lexik_jwt_authentication.jwt_token_authenticator
```

