# Tester son API de A à Z grâce à Behat

## Connaitre ce que l'on veut tester

Pour mes tests fonctionnels API, je ne voulais pas simplement tester les entrées et sorties des rêquetes, mais TOUT le processus incluant : 
- Les entrées (les différents types de données envoyés)
- Les sorties (les différents formats, les cas d'erreur)
- Ce qui doit se trouver dans la base de données après le retour API
- Ce qui doit se trouver dans les files RabbitMQ (messenger)
- Ce qui doit se trouver dans les logs 

Basiquement je veux ~100% de code coverage sur mes controllers API

## Behat

Behat est un framework PHP utilisé pour faire du BDD (Behaviour Driven Development). L'idée est que le métier et les devs partagent un langage commun pour mieux comprendre comment fontionne l'application.
Behat utilise **Gherkin** comme syntaxe pour ses tests.

Chaque ligne correspond à une définition du test et/ou une action qui se passe.

L'architecture générale d'un fichier de test (appelé **feature**) est la suivante :
```gherkin 
Feature : Ici un court texte explicatif de ce qu'on va tester dans le fichier
    In order to ...
    As ....
    I want to ... 

    Scenario : Une petite explication du cas de test
        Given une précondition (ex : présence d'un objet en BDD)
        And une autre précondition
        When une action de l'utilisateur de l'API (typiquement un call sur une route)
        And une autre action
        Then quelque chose que je dois tester
        And autre chose que e dois tester
```
Dans le cas du test d'une API, nous n'utiliserons pas forcément un utilisateur physique, ni des steps "client" / "fonctionnels" (End to end) mais plutot des tests "techniques". A noter que Behat supporte le français si vous preférez rédiger vos tests dans la langue de Molière.

Arborescence des fichiers / De quoi avons nous besoin :
- 1 fichier de configuration nommé ***behat.yml*** situé à la racine de votre application.
- Des fichiers ***.feature***. Idéalement un par feature différente a tester (ex : un endpoint). Ces fichiers sont la plupart du temps rangés dans le dossier ***features*** à la racine de votre application.
- Des fichiers ***context*** qui vous permettront d'implémenter vos "steps" afin qu'ils fassent ce que vous souhaitez. C'est la que la magie opère. La plupart du temps ils se trouvent dans le répertoire ***tests/Behat***. Les fichiers contextes peuvent être mutualisés pour les tâches que vous aurez à répéter dans plusieurs features. Attention cependant, l'ordre dans lequel vous les appelez est important car c'est l'ordre dans lequel ils seront executés. 

## Installation
```shell
composer require --dev friends-of-behat/symfony-extension
composer require --dev dvdoug/behat-code-coverage (non obligatoire)
```

## Etude d'un cas concret
Pour illustrer les propos de cet article, nous allons partir sur un cas concret : une API qui gère les données d'un jeu de société. L'exemple est volontairement simpliste afin de ne pas perdre de temps en explications qui ne concernent pas notre sujet.

Imaginons le schéma de base de données suivant :
![MCD](../img/behat_boardgame_mcd.png?raw=true "Read and write rights")

Nous allons donc avoir un endpoint qui va renvoyer les données suivantes [GET]: 
```json
{
  "name" : "Ticket to ride USA",
  "nbPlayers" : "2-5",
  "minAge" : 7,
  "year" : 2004,
  "author" : {
    "firstname" : "Alan",
    "lastname" : "Moon"
  },
  "category" : {
    "name" : "strategy"
  },
  "mechanics" : [
      { "name" : "Drafting" },
      { "name" : "Hand Management" },
      { "name" : "Network and Route Building" },
      { "name" : "Set Collection" }
    ] 
}
```

Et un endpoint qui va sauvegarder ces données en base de données et qui enverra en asynchrone un message pour dire qu'un nouveau jeu a été ajouté [POST].

### Test du endpoint de récupération d'un jeu [GET]

Les différentes valeurs de retour :
- [200] Renvoie notre jeu de société (ci-dessus)
- [404] Le jeu n'a pas été trouvé + log

Notre Gherkin ressemblera donc à ceci :
```gherkin
Feature Get a boardgame from database
  In order to get a boardgame
  As an API user
  I want to send a query with the game ID and get the game info

  Background: 
    Given I have a game in my database with ID 1

  Scenario: The requested Id does not exist
    When I request http://boardgame.test/9999 using HTTP method "GET" 
    Then the status code must be 404
    And the response should contain
      | error             | Game not found             |
    And the logger should contain a log of type "info" with content "Game with id 9999 not found"

  Scenario: The game exists and is returned by the API
    When I request http://boardgame.test/1 using HTTP method "GET"
    Then the status code must be 200
    And the response should contain : 
      | name              | Ticket to ride USA         |
      | nbPlayers         | 2-5                        |
      | minAge            | 7                          |
      | year              | 2004                       |
      | author.firstname  | Alan                       |
      | author.lastname   | Moon                       |
      | category.name     | strategy                   |
      | mechanics[0].name | Drafting                   |
      | mechanics[1].name | Hand Management            |
      | mechanics[2].name | Network and Route Building |
      | mechanics[3].name | Set Collection             |
```

Pour faire fonctionner notre fichier feature, nous allons creer plusieurs contextes : 
- 1 fichier de contexte pour gérer la base de données
- 1 fichier de contexte pour gérer les requetes HTTP
- 1 fichier de contexte pour gérer le systeme (ici les logs)

Ces fichiers seront partagés avec l'autre endpoint (post) car pour le moment ils sont assez génériques. Si nous avions à créer des requetes plus spécifiques ou gérer plusieurs BDD alors nous ferions de l'héritage de context pour mutualiser au maximum les informations. 

```php
// tests/Behat/RequestContext

<?php

declare(strict_types=1);

namespace App\Tests\Behat;

use Behat\Gherkin\Context\Context;
use Behat\Gherkin\Node\TableNode;
use PHPUnit\Framework\Assert;
use Symfony\Component\HttpFundation\Request;
use Symfony\Component\HttpFundation\Response;
use Symfony\Component\HttpKernel\Kernel;
use Symfony\Component\HttpKernel\KernelInterface;
use Symfony\Component\PropertyAccess\PropertyAccess;

class RequestContext implements Context
{
  protected KernelInterface $kernel;

  protected Response $response;

  public function __construct(KernelInterface $kernel)
  {
    $this->kernel = $kernel;
  }

  /**
   * @When I request :path using HTTP method :method 
   */
  public function iRequestUsingHttpMethod(
    string $path, 
    string $method
    ) : void {
        $request = Request::create($path, $method);
        $this->response = $this->kernel->handle($request);

        // si vous avez des evenements dans le kernel terminate, ne pas oublier ces lignes ;) 
        Assert::assertInstanceOf(Kernel::class, $this->kernel);
        $this->kernel->terminate($request, $this->response);
    }

    /**
     * @Then the status code must be :statusCode
     */
    public function theStatusCodeMustBe(int $statusCode): void
    {
       $resStatusCode = $this->response->getStatusCode();

       Assert::AssertEquals($statusCode, $resStatusCode);
    }

    /**
     * @Then the response should contain
     */
    public function theResponseShouldContainTableNode(TableNode $tableNode): void
    {
       Assert::assertIsString($this->response->getContent());
       $content = json_decode($this->response->getContent());

       // https://symfony.com/doc/current/components/property_access.html
       $propertyAccessor = PropertyAccess::createPropertyAccessor();

       foreach ($tableNode->getRowsHash() as $fieldName => $value) {
         Assert::assertEquals($value, $propertyAccessor->getValue($content, $fieldName));
       }
    }
}
```

```php
// tests/Behat/DatabaseContext

<?php

declare(strict_types=1);

namespace App\Tests\Behat;

use App\Entity\Author;
use App\Entity\Category;
use App\Entity\Game;
use App\Entity\Mechanic;
use Behat\Gherkin\Context\Context;
use Doctrine\ORM\EntityManagerInterface;
use Doctrine\ORM\Mapping\ClassMetadata;
use Doctrine\ORM\Tools\SchemaTool;

class DatabaseContext implements Context
{
  private EntityManagerInterface $entityManager;

  public function __construct(EntityManagerInterface $entityManager)
  {
    $this->entityManager = $entityManager;
  }

  /**
   * @BeforeScenario 
   */
  public function resetDatabase(): void
  {
    $metadata = $this->entityManager->getMetadataFactory()->getAllMetadata();

    if (false === empty($metadata))
    {
      $tool = new SchemaTool($this->entityManager);
      $tool->dropSchema($metadata);
      $tool->createSchema($metadata);
    } else {
      throw new ShemaException('No metadata classes to process.');
    }
  }

  /**
   * @Given I have a game in my database with ID :id
   */
  public function iHaveAGameInDb(int $id): void
  {
    // ici on ne se sert pas de l'id, les jeux seront insérés dans l'ordre et comme la base est recrée à chaque test, on commencera toujours par 1. C'est surtout dans le cas où on en genererait plusieurs avec des caractéristiques différentes afin de s'y retrouver dans le gherkin
    $game = $this->generateGame();
      
    $this->entityManager->persist($game);
    $this->entityManager->flush();
  }

 /**
  * Utile si vous avez plusieurs cas où vous avez besoin de générer un jeu ... Puis de le modifier par exemple en utilisant un tableNode avec les différentes propriétés à changer
  */
  private function generateGame(): Game
  {
    $author = (new Author())
      ->setFirstName('Alan')
      ->setLastName('Moon')
    ; 

    $category = new Category('Strategy');

    $mechanic1 = new Mechanic('Drafting');
    $mechanic2 = new Mechanic('Hand Management');
    $mechanic3 = new Mechanic('Network and Route Building');
    $mechanic4 = new Mechanic('Set Collection');

    $game = (new Game())
      ->setName('Ticket to ride USA')
      ->setNbPlayers('2-5')
      ->setMinAge(7)
      ->setYear(2004)
      ->setAuthor($author)
      ->setCategory($category)
      ->addMechanic($mechanic1)
      ->addMechanic($mechanic2)
      ->addMechanic($mechanic3)
      ->addMechanic($mechanic4)
    ;

    return $game;
  }
}
```
```php
// tests/Behat/SystemContext

<?php

declare(strict_types=1);

namespace App\Tests\Behat;

use Behat\Gherkin\Context\Context;
use Behat\Gherkin\Node\TableNode;
use Monolog\Handler\TestHandler;
use Monolog\Logger;
use PHPUnit\Framework\Assert;
use Symfony\Component\HttpKernel\KernelInterface;

class SystemContext implements Context
{

  private const LOG_LEVELS = [
    'info' => 200,
    'warning' => 300,
    'error' => 400
  ];

  protected KernelInterface $kernel;

  public function __construct(KernelInterface $kernel)
  {
    $this->kernel = $kernel;
  }

  /**
   * @Then the logger should contain a log of type :type with content :message 
   */
  public function theLoggerShouldContainALogOfTypeWithContent(string $type, string $message): void
  {
    /** @var TestHandler $testHandler */
    $testHandler = null;

    $logger = $this->kernel->getContainer()->get('monolog.logger.tests');
    Assert::assertNotNull($logger);

    foreach ($logger->getHandlers() as $handler) {
      if ($handler instanceof TestHandler) {
        $testHandler = $handler;
      }
    }

    Assert::assertNotNull($testHandler);
    Assert::assertTrue($testHandler->hasRecordThatContains(
      $message,
      self::LOG_LEVELS[$type]
    ));
  }
}
```

Nous avons notre fichier de tests et nos contextes pour le faire fonctionner. Il ne nous reste plus qu'a configurer nos services pour utiliser le logger de test et nous sommes prêts.
Pour cela 2 fichiers sont à configurer : 
```yaml
#config/packages/test/monolog.yaml

monolog:
  channels: ["tests"]
  handlers:
  ......
    tests:
      type: test
      level: debug
      channels: ["tests"]
```

```yaml
#config/services_test.yaml

App\Controller\GameController:
  public: true
  bind:
    $logger: '@monolog.logger.tests'
```

Ca y est nous avons de quoi tester notre premier endpoint, la récupération de nos jeux. Nous allons maintenant passer à notre 2e endpoint : la création d'un nouveau jeu.