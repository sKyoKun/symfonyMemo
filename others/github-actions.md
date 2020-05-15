# Github actions

Github actions can be triggered on each event (push, pull request ...).
It can be part of a CI to launch the unit tests suites or do some code cleaning (with cs cleaner for example)

Here is how to put it to work with a symfony project. First create a ".github" folder with a "workflows" subfolder. 
Next, all our new actions will be created with yaml files.

## Launch unit tests on each push

```yaml
name: CI - run tests

on: [push]

jobs:
  build:

    runs-on: ubuntu-latest # which distribution will be used

    steps: # steps will execute each commands under the run key
    - uses: actions/checkout@v1
    - name: Setup Symfony env
      run: cp .env.example .env
    - name: Install Composer Dependencies
      run: composer install
    - name: Create JWT keys
      run: | # if you have more than one command to do in a single step, use a pipe
        mkdir config/jwt
        openssl genrsa -passout pass:password -out config/jwt/private.pem -aes256 4096
        openssl rsa -passin pass:password -pubout -in config/jwt/private.pem -out config/jwt/public.pem
    - name: Setup TEST env
      run: cp .env.test .env
    - name: Create database
      run: |
        mkdir var/data
        bin/console doctrine:database:create --env=test
        bin/console doctrine:schema:create --env=test
    - name: Launch unit tests 
      run: bin/phpunit
```

## Fix files on pull request with CSFixer

```yaml
name: CI - CS Fixer

on: [pull_request]

jobs:
  build:

    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v1
      - name: Setup Symfony env
        run: cp .env.example .env
      - name: Install Composer Dependencies
        run: composer install
      - name: Run PHPCSFIXER
        run: vendor/bin/php-cs-fixer fix --verbose --config=php_cs.php
      - name: Commit changed files
        uses: stefanzweifel/git-auto-commit-action@v2.5.0 # a tierce bundle action for github. Automatically commit changed files
        with:
          commit_message: Apply php-cs-fixer changes
          branch: ${{ github.head_ref }} # current branch
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }} # GITHUB_TOKEN is automatically created for the workflow
```

## Create a branch with the code coverage on a schedule

Before running the code coverage, another script deleting de code-coverage branch must be scheduled.

```yaml
name: CI - run tests with code coverage and push on branch code-coverage

on:
  schedule:
    # * is a special character in YAML so you have to quote this string
    - cron:  '30 11,22 * * *'

jobs:
  build:

    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v1 # we checkout on our code-coverage branch
        with:
          ref: code-coverage
      - name: Remove code coverage folder (build)
        run: rm -rf build
      - name: Commit changed files
        uses: stefanzweifel/git-auto-commit-action@v2.5.0
        with:
          commit_message: Remove code coverage
          branch: code-coverage
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

```yaml
name: CI - run tests with code coverage and push on branch code-coverage

on:
  schedule:
    # * is a special character in YAML so you have to quote this string
    # that cron is launched at 12 and 23 GMT
    - cron:  '0 12,23 * * *'

jobs:
  build:

    runs-on: ubuntu-latest

    steps:
      # first setup the project with test env
      - uses: actions/checkout@v1
      - name: Setup Symfony env
        run: cp .env.example .env
      - name: Install Composer Dependencies
        run: composer install
      - name: Create JWT keys
        run: |
          mkdir config/jwt
          openssl genrsa -passout pass:password -out config/jwt/private.pem -aes256 4096
          openssl rsa -passin pass:password -pubout -in config/jwt/private.pem -out config/jwt/public.pem
      - name: Setup TEST env
        run: cp .env.test .env
      - name: Create database
        run: |
          mkdir var/data
          bin/console doctrine:database:create --env=test
          bin/console doctrine:schema:create --env=test
      # launch tests with code coverage (will create a new directory "build")
      - name: Launch unit tests with code coverage
        run: bin/phpunit --coverage-html build/
      # then commit the files on a specific branch (code-coverage) 
      - name: Commit changed files
        uses: stefanzweifel/git-auto-commit-action@v2.5.0
        with:
          commit_message: Generate code coverage
          branch: code-coverage
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```