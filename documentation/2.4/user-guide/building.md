---
---
# Building from Source <a name="building-from-source"/>

These instructions work for each of the modules, except for JMS Replication, which requires installation of a message queue. See that module for details.

 


## Building an Ehcache distribution from source <a name="building-an-ehcache-distribution-from-source"/>

To build Ehcache from source:

1. Check the source out from the subversion repository.
1. Ensure you have a valid JDK and Maven 2 installation.
1. From within the ehcache/core directory, type `mvn -Dmaven.test.skip=true install`


## Running Tests for Ehcache <a name="running-tests-for-ehcache"/>

To run the test suite for Ehcache:

1. Check the source out from the subversion repository.
1. Ensure you have a valid JDK and Maven 2 installation.
1. From within the ehcache/core directory, type `mvn test`
1. If some {performance tests fail}, add a `-D net.sf.ehcache.speedAdjustmentFactor=x` System property to your command line, where x is how many times your machine is slower than the reference machine. Try setting it to 5 for a start.


## Deploying Maven Artifacts

Ehcache has a repository and snapshot repository at oss.sonatype.org.
The repository is synced with the Maven Central Repository.

To deploy:

~~~
mvn deploy
~~~

This will fail because SourceForge has disabled ssh exec. You need to create missing directories manually using sftp access `sftp gregluck,ehcache@web.sourceforge.net`


## Building the Site <a name="building-the-site"/>

(These instructions are for project maintainers)
To build the site use:


~~~
mvn -Dmaven.test.skip=true package site
~~~

The site needs to be deployed from the target/site directory using:

~~~
rsync -v -r * ehcache-stage.terracotta.lan:/export1/ehcache.org
sudo -u maven -H /usr/local/bin/syncEHcache.sh
~~~


## Deploying a release


### Maven Release <a name="maven-release"/>

~~~
mvn deploy
~~~


### Sourceforge Release <a name="sourceforge-release"/>

~~~
mvn assembly:assembly
~~~

then manually upload to SourceForge with `sftp gregluck@frs.sourceforge.net` and complete the file release process