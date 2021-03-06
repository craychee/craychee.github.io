---
title: Testing and Deploying Sculpin to github pages on CircleCI
tags:
- automated testing
- automated deployment
- CI
- sculpin
categories:
    - sculpin
    - testing
description: Automate the deployment of your Sculpin project's artifact to Github Pages using CircleCI. Set it and forget it.
---
My setup here was adapted from [this excellent Wouter de Jong][wonter] post, which described automated deployment of sculpin using `travisci`, for [Circleci][circle] and testing with [behat][behat]. The original blog should absolutely be read first.

[circle]:https://circleci.com
[wonter]:http://wouterj.nl/2015/02/using-travis-to-build-your-sculpin-blog/
[behat]:http://docs.behat.org/en/v3.0/

Here is what my `circle.yml` for deploying this sculpin blog looks like:

    machine:
      php:
        version: 5.5.11
      hosts:
        craychee.local: 127.0.0.1

    dependencies:
      pre:
      - curl -O https://download.sculpin.io/sculpin.phar
      - php sculpin.phar install
      - cp circle.conf /etc/apache2/sites-available/default
      - sudo sed -e "s?%HOME%?$(pwd)?g" --in-place /etc/apache2/sites-available/default
      - sudo service apache2 restart
      post:
      - php sculpin.phar generate --env prod

    test:
      override:
          - bin/behat -f pretty

    deployment:
      production:
        branch: publish
        commands:
          - sh publish.sh


Once again, actually reading the [circleci documentation][documentation] will decrypt most of this for you, including variable names ("machine", "pre", "override", etc.).
[documentation]:https://circleci.com/docs/configuration

This is what I am doing differently than Wonter de Jung suggests.

**I am installing sculpin from source rather than installing with `composer.json`.**
~~~php
# circle.yml
...
    dependencies:
      pre:
      - curl -O https://download.sculpin.io/sculpin.phar
      - php sculpin.phar install
...
~~~
Installing with composer is probably more professional, at least it is more customizable, but I had no such need.

**I am pointing apache2 at my generated sculpin site.**

I am building the production directory `output_prod` and running acceptance tests on it, only deploying if they pass. In order to do this, I need to set up a server to serve up the site for behat to crawl.

~~~php
# circle.yml
...
    dependencies:
      pre:
      ...
      - cp circle.conf /etc/apache2/sites-available/default
      - sudo sed -e "s?%HOME%?$(pwd)?g" --in-place /etc/apache2/sites-available/default
      - sudo service apache2 restart
...
~~~

Here is what my generic `circle.conf` looks like:

    <VirtualHost *:80>
        UseCanonicalName Off
        DocumentRoot %HOME%/output_prod

      <Directory %HOME%/*>
        Options FollowSymLinks
        AllowOverride None
        Order allow,deny
        Allow from all
      </Directory>

    </VirtualHost>


The `%HOME%` variable was replaced inside the `circle.yml`:

~~~php
      - sudo sed -e "s?%HOME%?$(pwd)?g" --in-place /etc/apache2/sites-available/default
~~~

With all that setup, **I can execute my behat test suite** and deploy to github pages if all is well. I took entirely Wonter's recommendation for my `publish.sh`, only omitting the build of the `output_prod` directory.

One last thing: I didn't need to set up a github token on `circleci` for deployment. This is either a feature of circleCI or a security breach on my part.
