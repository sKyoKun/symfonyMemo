# Functional tests

## With Liip functional test bundle

Functional tests complete the unit tests by testing the controllers. We can simulate a client calling our routes. 
In this chapter, we'll see how we do it with [liip/functional-test-bundle](https://github.com/liip/LiipFunctionalTestBundle)

Like for the unit tests, it's recommended to cover all the possible cases (return values, error messages, exceptions ...)

Our test will cover the login to an API.

```php
<?php

namespace App\Tests;

use App\Entity\User;
use Liip\FunctionalTestBundle\Test\WebTestCase;
use Liip\TestFixturesBundle\Test\FixturesTrait;
use Symfony\Component\HttpFoundation\Response;

class LoginTest extends WebTestCase
{
    use FixturesTrait; // This trait helps us to load fixtures

    public function testLoginOK(): void
    {
        // Now we can use loadFixtureFiles. 
        // The first parameter is an array of fixtures files
        // The second parameter tells the test to erase the data that might be in the database before this test
        $fixtures = $this->loadFixtureFiles(
            [
                'fixtures/base.yml',
            ],
            true
        );

        /** @var User $user */
        $user      = $fixtures['base_user'];
        $client    = $this->makeClient();
        $container = $client->getContainer();

        // lets configure our request
        $client->request(
            'POST',
            $container->get('router')->generate('api_login_check'),
            [],
            [],
            ['CONTENT_TYPE' => 'application/json'],
            \json_encode([
                'username' => $user->getEmail(),
                'password' => 'user-test',
            ]) // our content
        );
        
        // we get the controller's returned content.
        $content = \json_decode($client->getResponse()->getContent(), true);

        // we can now test the values we want.
        $this->assertArrayHasKey('token', $content);
        $this->assertArrayHasKey('refresh_token', $content);
        $this->assertEquals(Response::HTTP_OK, $client->getResponse()->getStatusCode());
    }

    // Do not forget to test the KO cases (i.e bad credentials here)
    public function testLoginKO(): void
    {
        $fixtures = $this->loadFixtureFiles(
            [
                'fixtures/base.yml',
            ],
            true
        );

        $user      = $fixtures['base_user'];
        $client    = $this->makeClient();
        $container = $client->getContainer();
        $client->request(
            'POST',
            $container->get('router')->generate('api_login_check'),
            [],
            [],
            ['CONTENT_TYPE' => 'application/json'],
            \json_encode([
                'username' => $user->getEmail(),
                'password' => 'bad-password',
            ])
        );

        $this->assertEquals(Response::HTTP_UNAUTHORIZED, $client->getResponse()->getStatusCode());
    }
}

```