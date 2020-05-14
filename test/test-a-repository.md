# Test a repository

Sometimes you want to check that your repository function returns the right values. 
You can test them with fixtures (liip/test-fixtures-bundle + hautelook/alice-bundle), and a repository test : 

```php
// tests/Repository/UserRepositoryTest.php
namespace App\Tests\Repository;

use App\Entity\User;
use Symfony\Bundle\FrameworkBundle\Test\KernelTestCase;

// We extends KernelTestCase to have access to our entity manager
class UserRepositoryTest extends KernelTestCase
{
    use FixturesTrait;

    /**
     * @var EntityManager
     */
    private $entityManager;

    // Will be called before each test
    protected function setUp(): void
    {
        $kernel = self::bootKernel();

        $this->entityManager = $kernel->getContainer()
            ->get('doctrine')
            ->getManager();
    }

    // Will be called after each test
    protected function tearDown(): void
    {
        parent::tearDown();

        // doing this is recommended to avoid memory leaks
        $this->entityManager->close();
        $this->entityManager = null;
    }

    public function testGetTotalNumberOfUser()
    {
        // Here we load our fixtures that will contain 2 users in our database.
        $fixtures = $this->loadFixtureFiles(
            [
                'fixtures/Repository/UserRepository/get_user_total_number.yml',
            ],
            true
        );

       // We call our repository method
        $nbUsers = $this->entityManager
            ->getRepository(User::class)
            ->getTotalNumberOfUser();

        // We test what the result should be
        $this->assertEquals(2, $nbUsers);
    }
}
```
