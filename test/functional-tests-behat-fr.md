# Tester son API de A à Z grâce à Behat

## [Connaitre ce que l'on veut tester](#connaitre-ce-que-lon-veut-tester)

Pour mes tests fonctionnels API, je ne voulais pas simplement tester les entrées et sorties des rêquetes, mais TOUT le processus incluant : 
- Les entrées (les différents types de données envoyés)
- Les sorties (les différents formats, les cas d'erreur)
- Ce que la BDD doit contenir comme données
- Ce que les files RabbitMQ (messenger) doivent contenir comme messages
- Ce que les logs doivent contenir comme messages

Basiquement je veux ~100% de code coverage sur mes controllers API

## [Behat](#behat)

Behat est un framework PHP utilisé pour faire du BDD (Behaviour Driven Development). L'idée est que le métier et les devs partagent un langage commun pour mieux comprendre comment fontionne l'application.
Behat utilise **Gherkin** comme syntaxe pour ses tests.

Chaque ligne correspond à une définition du test et/ou une action qui se passe.

L'architecture générale d'un fichier de test (appelé **feature**) est la suivante :

<script src="https://gist.github.com/sKyoKun/36f65b5135d0acf6ebea4fc1f47e0f6d.js"></script>

Dans le cas du test d'une API, nous n'utiliserons pas forcément un utilisateur physique, ni des steps "client" / "fonctionnels" (End to end) mais plutot des tests "techniques". A noter que Behat supporte le français si vous preférez rédiger vos tests dans la langue de Molière.

Arborescence des fichiers / De quoi avons nous besoin :

- 1 fichier de configuration nommé ***behat.yml*** situé à la racine de votre application.
- Des fichiers ***.feature***. Idéalement un par feature différente a tester (ex : un endpoint). Ces fichiers sont la plupart du temps rangés dans le dossier ***features*** à la racine de votre application.
- Des fichiers ***context*** qui vous permettront d'implémenter vos "steps" afin qu'ils fassent ce que vous souhaitez. C'est la que la magie opère. La plupart du temps ils se trouvent dans le répertoire ***tests/Behat***. Les fichiers contextes peuvent être mutualisés pour les tâches que vous aurez à répéter dans plusieurs features. Attention cependant, l'ordre dans lequel vous les appelez est important car c'est l'ordre dans lequel ils seront executés. 

## [Installation](#installation)

```shell
composer require --dev friends-of-behat/symfony-extension
composer require --dev dvdoug/behat-code-coverage (non obligatoire)
```

## [Etude d'un cas concret](#etude-dun-cas-concret)

Pour illustrer les propos de cet article, nous allons partir sur un cas concret : une API qui gère les données d'un jeu de société. L'exemple est volontairement simpliste afin de ne pas perdre de temps en explications qui ne concernent pas notre sujet.

Imaginons le schéma de base de données suivant :
![MCD](../img/behat_boardgame_mcd.png?raw=true "Read and write rights")

Nous allons donc avoir un endpoint qui va renvoyer les données suivantes [GET]: 

<script src="https://gist.github.com/sKyoKun/4207f9c0a0e91f48f9b46c75b19e839a.js"></script>

Et un endpoint qui va sauvegarder ces données en base de données et qui enverra en asynchrone un message pour dire qu'un nouveau jeu a été ajouté [POST].

### [Test du endpoint de récupération d'un jeu (GET)](#test-du-endpoint-de-récupération-dun-jeu-get)

Les différentes valeurs de retour :

- [200] Renvoie notre jeu de société (ci-dessus)
- [404] Le jeu n'a pas été trouvé + log

Notre Gherkin ressemblera donc à ceci :

<script src="https://gist.github.com/sKyoKun/bcc1520f7304791a74aec70d37951851.js">
</script>

Pour faire fonctionner notre fichier feature, nous allons creer plusieurs contextes : 

- 1 fichier de contexte pour gérer la base de données
- 1 fichier de contexte pour gérer les requetes HTTP
- 1 fichier de contexte pour gérer le systeme (ici les logs)

Ces fichiers seront partagés avec l'autre endpoint (post) car pour le moment ils sont assez génériques. Si nous avions à créer des requetes plus spécifiques ou gérer plusieurs BDD alors nous ferions de l'héritage de context pour mutualiser au maximum les informations.

<script src="https://gist.github.com/sKyoKun/1f3a730e1ea7ee8b1a4ba0efcc1a8a4a.js"></script>

<script src="https://gist.github.com/sKyoKun/718f6ff900aaea780c1fd047781d4dc0.js"></script>

<script src="https://gist.github.com/sKyoKun/409ef1f64fd5d3bc4238c2ce018ba41c.js"></script>

Nous avons notre fichier de tests et nos contextes pour le faire fonctionner. Il ne nous reste plus qu'à configurer nos services pour utiliser le logger de test et nous sommes prêts.
Pour cela 2 fichiers sont à configurer : 

<script src="https://gist.github.com/sKyoKun/4a8f8dbb44c041527f19797b4d27a98d.js"></script>

```yaml
#config/services_test.yaml

App\Controller\GameController:
  public: true
  bind:
    $logger: '@monolog.logger.tests'
```

Ca y est nous avons de quoi tester notre premier endpoint, la récupération de nos jeux. Nous allons maintenant passer à notre 2e endpoint : la création d'un nouveau jeu.

### [Test du endpoint de création d'un jeu (POST)](#test-du-endpoint-de-création-dun-jeu-post)

Les différentes valeurs de retour :
- [200] Renvoie notre jeu de société + envoi de notre message
- [400] Une valeur n'est pas valide + log

Notre Gherkin ressemblera à ceci :

<script src="https://gist.github.com/sKyoKun/3a54fed2fdbc8ae0427a3d5ab9f37bc9.js"></script>

Nous avons déjà vu les valeurs de retour que nous avons déjà précédemment implémentées. Nous n'allons que rajouter ce dont nous avons besoin pour traiter ces nouvelles étapes.

<script src="https://gist.github.com/sKyoKun/ad861b0f1d4606fcf0438238968f0831.js"></script>

```php
// tests/Behat/SystemContext

...

  public function __construct(
    private KernelInterface $kernelInterface,
    private TransportInterface $transport
    ) {
  }

  /**
   * @Then the messenger should contain the following message :
   */
  public function theMessengerShouldContainTheFollowingMessage(
    TableNode $expectedMessageAttributes
    ): void {
    /* @var Envelope[] $messagesFromTransport */
    $messagesFromTransport = $this->transport->get();
    $messageId = $expectedMessageAttributes->getRowsHash()['message.id'];

    // ici je filtre mes messages grâce à l'ID mais vous pouvez utiliser n'importe quelle valeur unique de votre model
    $correspondingMessages = array_values(
      array_filter(
        $messagesFromTransport, 
        fn ($enveloppe) => $messageId === $enveloppe->getMessage()->getId();
      )
    );

    Assert::assertCount(1, $correspondingMessages);
    $message = array_shift($correspondingMessages);
    Assert::assertInstanceOf(Enveloppe::class, $message);
    $amqpStamp = $message->last(AmqpStamp::class);
    Assert::assertInstanceOf(AmqpStamp::class, $amqpStamp);

    foreach ($expectedMessageAttributes->getRowsHash() as $name => $value) {
      // message values
      if (strstr($name, 'message')) {
        Assert::assertEquals(
          $value,
          (string) $message->getMessage()->{'get' . ucfirst(substr($name, 8))}
        );
      }
      // stamp values (si on rajoute des informations dans les headers AMQP de nos messages, sinon ignorer cette partie)
      if (strstr($name, 'amqp.')) {
        // si notre stamp contient des headers
        if (strstr($name, 'amqp.headers.')) {
          $headerName = substr($name, 13);

          // on check que le header existe
          Assert::assertArrayHaskey(
            $headerName, 
            $amqpStamp->getAttributes()['headers']
          );

          // si en plus on veut vérifier sa valeur ...
          if ('' !== trim($value)) {
            Assert::assertEquals($value, $amqpStamp->getAttributes()['headers'][$headerName]);
          } 
        }else {
            $attributeName = substr($name, 5);

            Assert::assertArrayHaskey(
              $attributeName, 
              $amqpStamp->getAttributes()
            );
            if ('' !== trim($value)) {
            Assert::assertEquals($value, $amqpStamp->getAttributes()[$attributeName]);
          }
          }
        }
      }
    }
  }
...

```

Configurons notre messenger pour qu'il n'envoie pas de message vers une queue RabbitMQ mais plutot qu'il soit stocké en mémoire
```yaml
#config/packages/test/messenger.yaml

framework:
  messenger:
    reset_on_message: true
    transports:
      game: 'in-memory://'
```

```yaml
#config/services_test.yaml

services:
  App\Tests\Behat\SystemContext:
    bind:
      $transport: '@messenger.transport.game'
    public: true
```

<script src="https://gist.github.com/sKyoKun/28ed4b5d950d773aa51b0fbe5fb9d900.js"></script>

## [Pour aller plus loin](#pour-aller-plus-loin)

### [Le code coverage](#le-code-coverage)

Le bundle que nous avons installé précémment (composer require --dev dvdoug/behat-code-coverage ) nous permet de bénéficier du code coverage. Pour cela il va analyser toutes les fichiers parcourus par nos tests fonctionnels et déterminer les lignes utiles parcourues et celles qui ne l'ont pas été. 

La configuration du bundle doit se faire dans le fichier *behat.yml*. La configuration complete se trouve sur le site internet de l'auteur : [https://behat.cc/en/stable/configuration.html](https://behat.cc/en/stable/configuration.html) 

Nous allons nous intéresser plus particulièrement aux options d'exclusion de fichiers et aux rapports de couverture. Notre fichier devient donc :

<script src="https://gist.github.com/sKyoKun/8ec998a49bbb27d362f95446a05a6c25.js"></script>

Ici nous avons décidé de ne tester que notre répertoire src/ contenant notre application. Parmi celui-ci, nous avons exclu Kernel car étant un fichier de Symfony, ce n'est pas lui que nous souhaitons couvrir. 

Maintenant que nous avons configuré notre bundle, voyons à quoi ressemblent les sorties :
![code coverage text](../img/code_coverage.png?raw=true "Code coverage text")
Ici nous avons la sortie console. Elle se résume à un récapitulatif des coverages par fichier avec un code couleur :

- rouge : Pas assez de couverture < 50%
- jaune : Couverture moyenne 50% < couverture <90%
- vert : Très bonne couverture > 90%

Ces codes paliers sont configurables également grâce au fichier behat.yml

Je trouve les couleurs par fichiers plus pertinentes que le résumé en haut qui peut parfois vraiment faire peur :) 

L'autre sortie que nous avons configuré est la version HTML. Pour les habitués de PHPUnit, l'interface est identique et permet de s'y retrouver facilement.
![code coverage html file list](../img/code_coverage_html.png?raw=true "Code coverage html file list")
Vous pouvez naviguer de fichiers en fichiers et vérifier les tests qu'il manque pour atteindre les 100% de couverture.
![code coverage html file detail](../img/code_coverage_html2.png?raw=true "Code coverage html file detail")

Note : Pour ceux qui utiliseraient des outils type Sonarqube pour la vérification de leur code, il est tout à fait possible d'exporter les rapports PHPUnit et Behat en clover pour qu'ils soient aggrégés et avoir le "véritable" code coverage de l'ensemble de votre application.