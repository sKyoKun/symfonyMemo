# Add data in JWT and retrieve it in our controllers

## Create a JWTCreated listener

Let's say we want to retrieve the company ID of the user : 

```php
// src/Event/Listener/JWTCreatedListener.php 
<?php

namespace App\Event\Listener;

use App\Entity\User;
use Lexik\Bundle\JWTAuthenticationBundle\Event\JWTCreatedEvent;

class JWTCreatedListener
{
    public function onJWTCreated(JWTCreatedEvent $event): void
    {
        /** @var $user User */
        $user = $event->getUser();

        $payload = \array_merge(
            $event->getData(),
            [
                'userId'  => $user->getId(),
                'companyId' => $user->getCompany()->getId(),
            ]
        );

        $event->setData($payload);
    }
}
```

## Create a JWTSent Subscriber
```php
// src/Event/Subscriber/JWTSentSubscriber.php 
<?php

namespace App\Event\Subscriber;

use App\Repository\CompanyRepository;
use Lexik\Bundle\JWTAuthenticationBundle\Services\JWTTokenManagerInterface;
use Symfony\Component\HttpKernel\Event\RequestEvent;
use Symfony\Component\Security\Core\Authentication\Token\AnonymousToken;
use Symfony\Component\Security\Core\Authentication\Token\Storage\TokenStorageInterface;
use Symfony\Component\Security\Core\Authentication\Token\TokenInterface;

class JWTSentSubscriber
{
    /** @var JWTTokenManagerInterface */
    private $jwtManager;

    /** @var TokenStorageInterface */
    private $tokenStorage;

    /** @var CompanyRepository */
    private $companyRepository;

    /**
     * @param JWTTokenManagerInterface $jwtManager
     * @param TokenStorageInterface    $tokenStorage
     * @param CompanyRepository        $companyRepository
     */
    public function __construct(JWTTokenManagerInterface $jwtManager, TokenStorageInterface $tokenStorage, CompanyRepository $companyRepository)
    {
        $this->jwtManager        = $jwtManager;
        $this->tokenStorage      = $tokenStorage;
        $this->companyRepository = $companyRepository;
    }

    /**
     * Hydrate objects from data in JWT Token and pass them to the Request.
     *
     * @param RequestEvent $event
     */
    public function onKernelRequest(RequestEvent $event): void
    {
        if ($this->userIsLoggedIn($this->tokenStorage->getToken())) {
            $request = $event->getRequest();
            $data    = $this->jwtManager->decode($this->tokenStorage->getToken());

            if (false === empty($data)) {
                if (isset($data['companyId'])) {
                    $company = $this->companyRepository->find($data['companyId']);
                    if (null !== $company) {
                        $request->attributes->set('company', $company);
                    }
                }
            }
        }
    }

    /**
     * Return if token storage contains a logged in user.
     *
     * @param TokenInterface|null $token
     *
     * @return bool
     */
    private function userIsLoggedIn(?TokenInterface $token): bool
    {
        if (false === $token instanceof AnonymousToken && null !== $token) {
            return true;
        }

        return false;
    }
}

```

## Add our listener and subscriber to our services.yml
```yaml
services:
    App\Event\Listener\JWTCreatedListener:
        tags:
            - { name: kernel.event_listener, event: lexik_jwt_authentication.on_jwt_created, method: onJWTCreated }

    App\Event\Subscriber\JWTSentSubscriber:
        tags:
            - { name: kernel.event_listener, event: kernel.request, method: onKernelRequest }
```

## Use it in a Controller !
```php
// src/Controller/MyController.php

class MyController 
{
    public function getCompanyInfo(Request $request, Company $company)
    {
        // Here $company will automatically be hydrated with the user company
    }
}
```
