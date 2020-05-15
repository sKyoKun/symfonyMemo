# Fixtures with Alice bundle and faker

Fixtures can help you set up a database for your tests with fake data. You can either put specific values 
in your fixtures or use Faker (https://github.com/fzaninotto/Faker) to put random data. 

Let's see how Alice fixtures work. 

```yaml
App\Entity\Order:
  order_1:
    price: '12.34' # here we set our price, we want to test it for a specific reason
    user: '@base_user' # a link to another object
    dateTime: <(new DateTime("2019-12-09T05:00:00+00:00"))>
  
App\Entity\User:
  base_user: # here we use faker to create an email/firstname/lastname
    email: '<email()>' 
    password: <('$2y$10$D2sLLqRQfZTf7CuaSQcmteM2c.Silu2RMLR/wp7FJOnmQGhr3KYOi')> # here we put an encoded password 
    firstName: '<firstName()>'
    lastName: '<lastName()>'
    orders: ['@order_1'] # a collection
```

Don't forget to put your relations on both sides for many2many. 

There are others specific keywords than can be used in fixtures :

```yaml
# construct helps you to initialize your object with values (in case you don't have getters or setters)
__construct: [<firstname()>, <lastname()>, <email()>]

# calls helps you when your method is not well named for example. 
__calls:
      - addOrdersToUser: ['@order_1']
```