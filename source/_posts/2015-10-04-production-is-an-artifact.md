---
title: Production is an Artifact of Development
subtitle: Ship Artifacts, not development tools.
tags:
- content strategy
categories:
description: Ship artifacts, not development tools. I give a brief history of the evolution of Drupal's project repository and provide my manifesto for the separation of production and development code.
---

**Note: This article is adapted from a presentation I gave at [NYCCamp](http://nyccamp.org/). It was improved greatly by the thoughtful feedback from my former colleagues: Larry Garfield, Steve Persch, Bec White, and Arthur Foelsche.**

##TL;DR
This is what development looks like:  
![Development conveyor belt](/img/tools_chain.png)

This is what a production artifact looks like:  
![Shipping box](/img/shipment_box_product.png)

This is what development markup looks like:

```sass
section {
    height: 100px;
    width: 100px;

    .class-one {
      height: 50px;
      width: 50px;

      .button {
        color: #074e68;
       }
     }
}
```

This is what a production artifact of that markup looks like:

```css
section {
  height: 100px;
  width: 100px;
}

section .class-one {
  height: 50px;
  width: 50px;
}

section .class-one. button {
  color: #074e68;
}
```

This is what your Drupal project's development repository might look like:

```
Vagrantfile
behat.yml
circle.yml
composer.json
composer.lock
PROJECT.info
PROJECT.module
README.md
build/
features/
cnf/
bin/
www/
vendor/
modules/
themes/
```

This is what your Drupal project's production artifact *should* look like:

```
CHANGELOG.txt
COPYRIGHT.txt
INSTALL.mysql.txt
INSTALL.pgsql.txt
INSTALL.sqlite.txt
INSTALL.txt
LICENSE.txt
MAINTAINERS.txt
README.txt
UPGRADE.txt
authorize.php
cron.php
includes/
index.php
install.php
misc/
modules/
profiles/
robots.txt
scripts/
sites/
themes/
update.php
web.config
xmlrpc.php
```

The tools that you use to build the thing are not the things that you should deploy. You deploy only the thing that your tools build.

I want to talk about how our Drupal projects' repositories got to the state they are in now and why our production repositories should be different.

##A Brief, Opinionated, Subjective History of a Shippable Drupal Project
For most of Drupal's life, almost certainly up until Drupal 7 was released, this is what a Drupal deployable looked like:

```
CHANGELOG.txt
COPYRIGHT.txt
INSTALL.mysql.txt
INSTALL.pgsql.txt
INSTALL.sqlite.txt
INSTALL.txt
LICENSE.txt
MAINTAINERS.txt
README.txt
UPGRADE.txt
authorize.php
cron.php
includes/
index.php
install.php
misc/
modules/
profiles/
robots.txt
scripts/
sites/
themes/
update.php
web.config
xmlrpc.php
```
***AND* a database:**

```
my_db.sql.gz
```

This is, for the unintiated, the root of Drupal's filesystem and a copy of the database. Within the Drupal root, there were a series of contributed modules located somewhere within the Drupal filesystem and almost certainly a few custom modules and/or libraries and a bunch of css files and templates. The specifics around which of these modules were enabled and how they were configured and what theme to use was information that was stored inside the database. In order to deploy Drupal satisfactorily, both the codebase and the database were required.

####Problem: Codebase was married to the database.
####Solution: The features module.
There were a lot of problems with a repository like the one described above. Aside from the "what database is the 'right' one?" problem (that is, what was the last copy of the database that contains all the expected changes) and the problems with working with multiple developers on one canonical database, clients could not make changes to their content while development was in progress. Worse, changes could not be rolled back and commits were meaningless without the database that cooresponded to them. 

Supporting a project deployed this way was impossible.

All sorts of silly things were done to get around this terrible mess. But the best and most widely adopted solution to this was to use the features module: exporting the configuration into code so that the database was not relied upon any longer to be the sole repository of this essential information.

Now a deployable Drupal repository might look something like this:

```
CHANGELOG.txt
COPYRIGHT.txt
INSTALL.mysql.txt
INSTALL.pgsql.txt
INSTALL.sqlite.txt
INSTALL.txt
LICENSE.txt
MAINTAINERS.txt
README.txt
UPGRADE.txt
authorize.php
cron.php
includes/
index.php
install.php
misc/
modules/
profiles/
robots.txt
scripts/
*sites/all/features*
themes/
update.php
web.config
xmlrpc.php
```

Similar but, crucially, no database.

####Problem: It was not clear when and how to apply changes to the database.
####Solution: Build scripts.

Eliminating the database from the deployment release cycle was a massive improvement. Content could be added and other information could be updated without any interruption. Deployments made changes to the database in code and could even then be rolled back.

But applying these changes to the database required that someone in the deployment process execute a series of commands. Typically these commands looked something like:

```
drush update-database
drush feature-revert all
drush clear-caches all
```
These commands ensured that the database was now in something at least closer to a known-state. Trouble started when the user relied upon to execute the code would forget a step (as humans are want to do) or the developers working in teams wouldn't apply these steps when developing code, meaning that they weren't working in known-states. In other words, we put the changes that needed to be made to the database in code but we didn't put instructions on how to apply those changes into code (update hooks got you closer but you still needed to apply them).

So the solution was a build script: some way of making the changes that developers were expecting explicit. But the place for this script was not inside the Drupal root. It didn't make sense to have the project root be the Drupal root anymore so another directory was added and a Drupal project began to look something like:

```
build-scripts/
drupal-root/
```

Now we had one directory for our Drupal root and one directory for scripts to act upon it.

####Problem: We weren't building on an environment that resembled production.
####Solution: Virtualization.
Now we were finally making our work explicit, consistent, and executable. And with the Drupal root now liberated from the project root, it was easier to see that a project's deliverable (a fully configured Drupal) was separate from the tools that created and supported that deliverable. We now felt free to add tools into the repository that were not part of the deliverable but would help us deliver great code.

One of the biggest pain points in development was that we weren't building in an environment that resembled production nor anything else. In fact, many of us weren't sure what our environment resembled. It was just whatever version of our LAMP (WAMP/XAMP) stack we happened to be on. This meant that not only did we have no assurance that our code would work on the environment to which it was destined but we also had no assurance that it would work on our colleagues' own environment.

We could resolve this through virtualization: define the destination system in code, ensure that all development is done inside that system. This is best done with [vagrant](https://docs.vagrantup.com/v2/). And so a `Vagrantfile` was thus added to our project's repository.

Now our project's repository looked something like:

```
Vagrantfile
build-scripts/
drupal-root/
```
Along with our Drupal root and instructions on how to enforce changes to the database with it, we defined the environment in which the project was meant to live.

####Problem: Testing Drupal took ages and provided little useful information.
####Solution: System testing.
Now that we could finally be assured that we were working on a known-state (specifically, a production environment), we could start being proactive about predicting how our code was going to behave there. Moreover, we could ensure that our code kept behaving as predicted throughout the development cycle.

This meant testing Drupal.

The topic of testing Drupal had been (and continues to be) one of the most excuses-ridden subjects: Drupal takes too long to test (I dare you to run all of the Simpletests inside Drupal core alone), Drupal is too tightly coupled to test properly, most shops are too small for testing to add value, it takes too long to write test, and so on. Drupal doesn't do itself any favors here but if we are going to continue to be professional Drupal developers, we must accept the limitations and find a path forward to build reliable, consistent, predictable code.

And so we use system tests.

System tests are high level tests run on the fully bootstrapped, integrated system to verify that the entire codebase meets some set of defined requirements. System tests are not unit tests. They aren't evaluating whether a piece of code outputs the expected results given a defined set of inputs. Rather, they are testing (much like a user would) if some action (usually by a user) yields the expected result. System tests are often done at the highest level possible: the browser.

Various tools were experimented with in our determination to test Drupal. Consensus is starting to form around one tool in particular: [Behat](http://docs.behat.org/en/v3.0/) and its [Drupal Extension](http://behat-drupal-extension.readthedocs.org/en/latest/intro.html).

Testing Drupal meant that we needed to host the requirements for the tooling to test Drupal as well as the tests themselves. For Behat, this meant a `composer.json` (and its coresponding `composer.lock`) to add Behat to the project, a `behat.yml` to configure Behat, as well as the tests themselves, which live inside a `features` directory.

And lo, a Drupal project now started to look like this:

```
behat.yml
composer.json
composer.lock
Vagrantfile
features/
build-scripts/
drupal-root/
```
At last, we had tests to define done and ensure that we stayed done throughout the project's lifecycle.

####Problem: We were lugging around both Drupal core and contributed modules inside our repository.
####Solution: Composer
The importance of adding a `composer.json` to a Drupal project to add a requirement like Behat cannot be overlooked. At the point that we started to add dependencies to the project via a dependency management tool rather than by committing the entire dependency to the project, we took a good hard look at what we were doing inside that Drupal root. What about all these contributed modules that we were using in our project? Aren't they dependencies? What about all the libraries and themes? What about Drupal core itself? Hadn't we all sworn a sacred oath never to touch these things? Why were we tempting fate?

Drush make had been around for a while but with Drupal 8 adopting composer and the usefulness of composer to add other helpful libraries (like guzzle or twig), why not use composer to manage components like Drupal contrib and Drupal core?

And so began the most important change in the history of a Drupal project: the Drupal root is no longer part of the project.

Instead, a Drupal project repository contains the dependency file (composer), features, custom code, and instructions on how to assemble the Drupal root from those parts. **The Drupal root is assembled from dependencies, build instructions, and the inclusion of custom code and configuration.**

A modern Drupal repository now looks something like:

```
custom/
themes/
behat.yml
composer.json
composer.lock
Vagrantfile
features/
build-scripts/
```

####Problem: Installing dependencies with Composer on a production environment is dangerous.
####Solution: Leverage a continuous integration server to build and deploy... a Drupal root codebase and a database.
Using composer to manage Drupal's dependencies is the way forward. It's sane. It's manageable. It's awesome. If you aren't ready to accept this for Drupal 7, you will have no choice with Drupal 8.

If I sound defensive it's because I am now about to list reasons why running `composer install` (which you need to do if you want all those juicy dependencies) is incredibly fragile and should never, ever be done on a production environment:

**Retrieving meta-data and dependencies eats a ton of ram.** Composer needs to go out and find the dependencies of your dependencies and the dependencies of those dependencies and so on until it has everything that everything needs. This can take a while and can be a major drain on resources.  
**Composer will fetch dependencies from a lot of sources.** Sources from all over the Internet, which can also change without notice, that you probably don't want to trust.  
**Rate limits on github.** Github will ask for authentication after hitting that limit. Ask your sysadmins how they feel about adding an authentication token to the production envionment.  
**Your deployment could take longer than expected.** For all the reasons above and many others, you may have planned on a 15 minute downtime to deploy your Drupal but that 15 minutes could easily turn into 30, 45, or worse.  
**Your dependencies could disappear.** You are referencing metadata inside composer and the truth is that many maintainers of projects aren't the greatest at semantic versioning. Your dependency could be removed without notice. 

For more thoughts on using Composer to install dependencies, I highly recommend [@ppetermann](https://twitter.com/ppetermann)'s insightful [blog post](https://devedge.wordpress.com/2015/05/16/a-few-thoughts-about-composer-and-how-people-use-it/).

Fortunately, there is a well-established method to assemble code inside a disposable environment: using a Continuous Integration server (CI server) such as [Travis](https://travis-ci.org/), [Circle](https://circleci.com/), or [Jenkins](https://jenkins-ci.org/). On an environment such as the ones that those providers create on a process such as a Pull Request or a merge with master, assembling the Drupal root can be done safely.

Using a CI server, you can put your deployment plan into code (which probably will include pulling down the current production database and making changes to it), ensure that it executed properly, and then deploy only the *artifact* of what your deployment created: in this case, the Drupal root. It makes no sense to deploy the composer files, the tests, the build script, the Vagrantfile, or any other helpers. Your Drupal root is your only shippable product. Deploy only that.

I describe this workflow in some detail here: [No Excuses: Automated Deployment](/blog/2015/08/08/no-excuses-part5-deployment/)

So now, while our *development*'s project repository might look like the above, our shippable Drupal project looks like:

```
CHANGELOG.txt
COPYRIGHT.txt
INSTALL.mysql.txt
INSTALL.pgsql.txt
INSTALL.sqlite.txt
INSTALL.txt
LICENSE.txt
MAINTAINERS.txt
README.txt
UPGRADE.txt
authorize.php
cron.php
includes/
index.php
install.php
misc/
modules/
profiles/
robots.txt
scripts/
sites/
themes/
update.php
web.config
xmlrpc.php
```
***AND* a database:**

```
my_db.sql.gz
```

Obligatory pretentious quote:
>We shall not cease from exploration, and the end of all our exploring will be to arrive where we started and know the place for the first time.
- T.S. Eliot

##Deploy only an artifact of your build process: a Manifesto

What I am advocating for here is a strict separation of Development and Production code. Development contains the tools and the parts to build the thing. Production contains only the thing.

Here is another provision: for the professional Drupal developer, Production is the first environment in which your client can access the project. Period. Not the environment that your client's client (their end user) access the project but *your* client. Your clients should only interact with production-ready code. This may be confusing because often the environment that your client's first begin interacting with the code is called *development* (as on Pantheon or Acquia) but for you, that's production. **Every deployment should be a full dress rehearsal.** It is your client's responsibility to decide from there when the show is ready to go live.

Here is my manifesto:  
####1. Development code should be readable as instructions for humans. Production code should be readable by a web service.
Make your development repository easily understood by human beings who are working on it. Make your production code easily read by the computer. Your production code is fully compiled before it hits production.

####2. We should depend on our platform hosts to host the thing, not the build the thing.
Our platform hosts do not need to support the building of an application. The building of the application should occur before the thing hits the host.

####3. Devs should be opinionated about development. Hosts should be opinionated about production.
Developers like us should form strong opinions (but hold them loosely) about how to deliver a project. We should continually seek tools and techniques to build and support our projects. And we should not rely on our hosts to make techniques available before we adopted them.  

On the other side: hosts should have strong opinions (held loosely) about how best to host the project. Hosts should refuse to support unsupported versions of packages. Hosts should make system decisions with a view to supporting sites in the wild. They shouldn't be fighting entropy by accommodating every developer's strong opinions. Rather, they should be enforcing their own.

####4. No dev dependency should be installed on a prod.  
No behat. No compass. No composer. Not ever.
