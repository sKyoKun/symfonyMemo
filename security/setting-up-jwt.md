# Setting up JWT

## Install the bundles 
```Shell
composer require "lexik/jwt-authentication-bundle"
```
If you need an extra security, you can use the refresh token bundle 
```Shell
composer "gesdinet/jwt-refresh-token-bundle"
```

## JWT Bundle 
### Generate the keys 
```Shell
mkdir config/jwt
openssl genrsa -passout pass:yourpass -out config/jwt/private.pem -aes256 4096
openssl rsa -passin pass:yourpass -pubout -in config/jwt/private.pem -out config/jwt/public.pem
```
**DO NOT COMMIT THE KEYS !!!**

### Add the keys in the .env
```yaml
###> lexik/jwt-authentication-bundle ###
JWT_SECRET_KEY=%kernel.project_dir%/config/jwt/private.pem
JWT_PUBLIC_KEY=%kernel.project_dir%/config/jwt/public.pem
JWT_PASSPHRASE=yourpass
###< lexik/jwt-authentication-bundle ###
```

### Add the env variable in the bundle config 
The bundle config can be found at : `config/packages/lexik_jwt_authentication.yaml`
```yaml
lexik_jwt_authentication:
    secret_key: '%env(resolve:JWT_SECRET_KEY)%'
    public_key: '%env(resolve:JWT_PUBLIC_KEY)%'
    pass_phrase: '%env(JWT_PASSPHRASE)%'
    token_ttl: 86400 # one day (then refresh token takes the next calls)
```

### Configure config/packages/security.yml
```yaml
firewalls:
  login:
      pattern:  ^/api/login
      stateless: true
      anonymous: true
      json_login:
          check_path:               /api/login_check
          success_handler:          lexik_jwt_authentication.handler.authentication_success
          failure_handler:          lexik_jwt_authentication.handler.authentication_failure

  api:
      pattern:   ^/api
      stateless: true
      guard:
          authenticators:
              - lexik_jwt_authentication.jwt_token_authenticator

access_control:
  - { path: ^/api/login, roles: IS_AUTHENTICATED_ANONYMOUSLY }
  - { path: ^/api,       roles: IS_AUTHENTICATED_FULLY }
```

### Configure config/routes.yml
```yaml
api_login_check:
    path: /api/login_check
```

## JWT Refresh bundle
### Configure config/packages/security.yml
```yml
firewalls:
  refresh:
      pattern:  ^/api/token/refresh
      stateless: true
      anonymous: true

access_control:
  - { path: ^/api/token/refresh, roles: IS_AUTHENTICATED_ANONYMOUSLY }
```
### Configure config/routes.yml
```yml
gesdinet_jwt_refresh_token:
    path:       /api/token/refresh
    controller: gesdinet.jwtrefreshtoken::refresh
```

### Update the schema
```Shell
php bin/console doctrine:schema:update --force

# or make and run a migration:
php bin/console make:migration
php bin/console doctrine:migrations:migrate
```

### Configure the bundle to use our own user provider
The configuration file can be found at : `config/packages/gesdinet_jwt_refresh_token.yaml`
```yml
gesdinet_jwt_refresh_token:
  ttl: 5184000 # 60 days (then user has to log in again)
  user_provider: security.user.provider.concrete.users # use our User class instead of the bundle's one
```
