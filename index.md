# Symfony Memo

This repo will contain all the things that we often have to do on a Symfony project.
As my memory is the same as Dory in "Finding Nemo", it'll help me save time not going on Google to look for everything ;) 

## Setting up the project

Let's set up a new Symfony skeleton project 
```Shell
composer create-project symfony/skeleton myAPIProject
```

### Useful bundles for an API

Then we require the bundles we may have to use in the following list : 

```json
"require": {
    "php": "^7.1.3",
    "ext-json": "*",
    "beberlei/doctrineextensions": "^1.2", # some good extensions for SQL queries 
    "doctrine/doctrine-bundle": "^1.11",
    "doctrine/doctrine-migrations-bundle": "^2.1",
    "doctrine/orm": "^2.6",
    "friendsofphp/php-cs-fixer": "^2.15", # clean code
    "gesdinet/jwt-refresh-token-bundle": "^0.8.3", # refresh a JWT token
    "hautelook/alice-bundle": "^2.5", # Used for tests
    "lexik/jwt-authentication-bundle": "^2.6",  # JWT authentication
    "nelmio/api-doc-bundle": "^3.4", # Api Documentation
    "sensio/framework-extra-bundle": "^5.5",
    "sensiolabs/security-checker": "^6.0",
    "symfony/apache-pack": "^1.0",
    "symfony/asset": "4.3.*",
    "symfony/console": "4.3.*",
    "symfony/dotenv": "4.3.*",
    "symfony/filesystem": "4.3.*",
    "symfony/flex": "^1.3.1",
    "symfony/form": "4.3.*",
    "symfony/framework-bundle": "4.3.*",
    "symfony/http-client": "4.3.*",
    "symfony/monolog-bundle": "^3.4",
    "symfony/security-guard": "4.3.*",
    "symfony/serializer": "4.3.*",
    "symfony/twig-pack": "^1.0",
    "symfony/validator": "4.3.*",
    "symfony/yaml": "4.3.*"
},
"require-dev": {
    "doctrine/doctrine-fixtures-bundle": "^3.3",
    "liip/functional-test-bundle": "^3.0.0", # Help for functional tests
    "liip/test-fixtures-bundle": "^1.0.0", # Load fixtures in functional tests
    "symfony/maker-bundle": "^1.14", # Not mandatory but a good help
    "symfony/test-pack": "^1.0",
},
```

## Security 
- [Setting up JWT on the project](security/setting-up-jwt.md)
- [Custom entity provider and firewalls in security.yml](security/custom-provider.md)
- [Enable login in swagger](security/login-in-swagger.md)
- [Add data in JWT after login and retrieve it from the controller](security/add-data-in-jwt.md)

## Playing with doctrine
- [Create and hydrate an object (not entity) from a database query](doctrine/hydrate-object-from-query.md)
- [Configure doctrine extensions to use advanced SQL commands](doctrine/extensions.md)

## Tests
- [Unit tests](test/unit-tests.md)
- [Fixtures with faker](test/fixtures-with-faker.md)
- [Functional tests with Liip functional bundle](test/functional-tests.md)
- [Functional tests with Behat / French version](test/functional-tests-behat-fr.md)
- [Testing a repository](test/test-a-repository.md)

## Swagger documentation
- Document the actions
- Use serialization groups in swagger models

## Others
- [Symfony & SQLServer (with and without doctrine)](others/symfony-sql-server.md)
- [Github actions (CI / CSFIXER)](others/github-actions.md)
