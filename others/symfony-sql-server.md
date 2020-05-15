# Symfony and SQL Server

## SQL Server setup on windows

### Download SQL Server + SQL Management  
     
[https://www.microsoft.com/fr-fr/sql-server/sql-server-downloads]()

[https://docs.microsoft.com/fr-fr/sql/ssms/download-sql-server-management-studio-ssms?view=sql-server-ver15]()

### SQL Server setup

Follow the default setup and then in SQL SERVER CONFIGURATION MANAGER, toggle “SQL Server Network Configuration” then Activate TCP/IP.  

![Activate TCP/IP](../img/sf_sqlserv_1.png?raw=true "Activate TCP/IP")

Then in SQL Server services > Restart the instance.

In SQL Server Management Studio : 

Right click on the server name > Security > Server authentication : SQL Server and Windows Authentication mode 

![SQL Server and windows authentication mode](../img/sf_sqlserv_2.png?raw=true "SQL Server and windows authentication mode")

* Create a database (Databases > clic droit > New database) 
* Toggle the tree and search Security > Users  
* Right click > new User and give it read and write rights.

![Read and write rights](../img/sf_sqlserv_3.png?raw=true "Read and write rights")

## PHP Configuration 

- PHP on windows 10 without wamp : https://davidperonne.com/installation-de-php-7-et-composer-sur-windows-10-en-mode-natif/  

- Install MSSQL extensions : https://pecl.php.net/package/sqlsrv/5.8.0/windows & https://pecl.php.net/package/pdo_sqlsrv/5.8.0/windows 
**WARNING** : Use 7.4 Non Thread Safe (NTS) x64 version !! 

- Unzip it in the ext folder of the latest PHP version 

- Add the extensions in the php.ini file

```ini
extension=pdo_sqlsrv 
extension=sqlsrv 
```

## Symfony configuration

### With doctrine
```yaml
# doctrine.yml

doctrine: 
    dbal: 
        driver: '%env(DATABASE_DRIVER)%' 
        host: '%env(DATABASE_HOST)%' 
        port: '%env(DATABASE_PORT)%' 
        dbname: '%env(DATABASE_NAME)%' 
        user: '%env(DATABASE_USER)%' 
        password: '%env(DATABASE_PWD)%'
```

```yaml
# .env.yml

DATABASE_DRIVER=pdo_sqlsrv
DATABASE_HOST=computer_name_or_host 
DATABASE_PORT=1433 
DATABASE_NAME=test 
DATABASE_USER=root 
DATABASE_PWD=password 
```

We can test that everything is OK with _bin/console doctrine:schema:validate -v_

### Without doctrine

Doctrine does not allow us to retrieve multiple rows for stored procedures. So we must use basic PDO queries to get procedures output.

```yaml
# .env.yml

DATABASE_DRIVER=sqlsrv
DATABASE_HOST=computer_name_or_host
DATABASE_PORT=1433
DATABASE_NAME=test
DATABASE_USER=root
DATABASE_PWD=password
```

```yaml
# services.yml

parameters:
    database_config:
        driver: '%env(DATABASE_DRIVER)%'
        host: '%env(DATABASE_HOST)%'
        port: '%env(DATABASE_PORT)%'
        user: '%env(DATABASE_USER)%'
        password: '%env(DATABASE_PWD)%'

    dbname: '%env(DATABASE_NAME)%'

services:
    App\Service\PDOHandler:
        class: App\Service\PDOHandler
        arguments:
            - '%database_config%'
```

```php
// App\Service\PDOHandler

<?php

namespace App\Service;

use PDO;

class PDOHandler
{
    /** @var string */
    private $driver;

    /** @var string */
    private $host;

    /** @var string */
    private $port;

    /** @var string */
    private $user;

    /** @var string */
    private $password;

    /**
     * @param array $databaseConfig
     */
    public function __construct(array $databaseConfig)
    {
        $this->driver   = $databaseConfig['driver'];
        $this->host     = $databaseConfig['host'];
        $this->port     = $databaseConfig['port'];
        $this->user     = $databaseConfig['user'];
        $this->password = $databaseConfig['password'];
    }

    /**
     * Returns a new PDO Instance for the database in parameter.
     *
     * @param string $databaseName
     *
     * @return PDO
     */
    public function getNewPDOInstance(string $databaseName)
    {
        $dns = $this->getDNSForDatabase($databaseName);

        return new PDO($dns, $this->user, $this->password);
    }

    /**
     * Returns the database DNS for PDO.
     *
     * @param string $databaseName
     *
     * @return string
     */
    private function getDNSForDatabase(string $databaseName)
    {
        return $this->driver . ':Server=' . $this->host . ',' . $this->port . ';Database=' . $databaseName;
    }
}
```