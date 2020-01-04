# Create and hydrate an object (not entity) from a query

Most of the time, we use entities to query and return data from the database.

It's sometime useful to hydrate another object instead of returning an entity collection or raw data from the database. We can do it really easily with doctrine. 

## Create the model
```php
<?php

namespace App\Model;

class MyClass
{
    /**
     * @var string
     */
    private $name;

    /**
     * @var string
     */
    private $concatenatedValue;
    
    public function __construct(string $name, string $concatenatedValue)
    {
        $this->name = $name;
        $this->concatenatedValue = $concatenatedValue;
    }
    
    public function setName(string $name): void
    {
        $this->name = $name;
    }
    
    public function getName(): string
    {
        return $this->name;
    }
    
   public function setConcatenatedValue(string $concatenatedValue): void
    {
        $this->concatenatedValue = $concatenatedValue;
    }
    
    public function getConcatenatedValue(): string
    {
        return $this->concatenatedValue;
    }
```

## Hydrate the model with query results 
```php
<?php

namespace App\Repository;

use Doctrine\Bundle\DoctrineBundle\Repository\ServiceEntityRepository;
use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\Common\Persistence\ManagerRegistry;

/**
 * Class MyClassRepository.
 */
class MyClassRepository extends ServiceEntityRepository
{
    /**
     * @param ManagerRegistry $managerRegistry
     */
    public function __construct(ManagerRegistry $managerRegistry)
    {
        parent::__construct($managerRegistry, MyClass::class);
    }
    
    public function getMyClasses(int $catId)
    {
        $result = $this->createQueryBuilder('m')
            ->select(
                "NEW App\Model\MyClass // this line is used to hydrate automatically our Model with the values inside the parenthesis
                (
                    m.name,
                    CONCAT(m.catId, '_', m.id), 
                )"
            )
            ->where('m.catid = :catId')
            ->setParameter('catId', $catId)
            ->getQuery()
            ->getResult();

        return new ArrayCollection($result);
    }
```
