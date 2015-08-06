---
layout: post
title:  "Installing Alfresco to Ubuntu 14.04 by hand"
author: emlyn
date:   2015-08-06 13:34:28
categories: alfresco ubuntu install 14.04 LTS
---

I've recently been tasked with working on a web application that
connects to [Alfresco 5.0.d](https://www.alfresco.com/). For that I
wanted to install Alfresco 5.0.d on a local Virtualbox VM that I could
SSH into and work and build against. This is my guide to do just
that. I'm doing this on Mac OSX 10.10, the instructions will probably
work for Linux also.

I've taken inspiration from the
[install script created by Loftux](https://github.com/loftuxab/alfresco-ubuntu-install)
and the ansible script created by
[libersoft](https://github.com/libersoft/ansible-alfresco). Thanks
chaps for putting those repos together!

Note, you can use the installation wizards from
[Alfresco](http://docs.alfresco.com/community/concepts/simpleinstalls-community-intro.html),
but who the heck is a fan of a massive bin file!? It ends up
installing packages outside of the apt-get package manager meaning
you'll manually need to patch all the dependencies should a security
flaw arise rather than relying on the exceptional work of Ubuntu and
Debian upstream. Capice?

## Base requirements

Download and install:

+ [VirtualBox](https://www.virtualbox.org/wiki/Downloads) - this
  creates and runs the Ubuntu virtual machine
+ [Vagrant](http://www.vagrantup.com/downloads.html) - this makes it
  easier to setup a virtual machine

## Installing Alfresco

Create a working folder in your home directory, create the
virtualmachine, ssh in:

### Create a Ubuntu 14.04 VM

I've created a small a github repo that contains a vagrantfile to
setup a virtualbox based on Ubuntu 14.04 with recommended settings
from Alfresco

{% highlight bash %}
git clone git@github.com:EmlynC/alfresco5-ubuntu-trusty.git
cd alfresco5-ubuntu-trusty
vagrant up
vagrant ssh
{% endhighlight %}

That'll take a little while to download and setup. Update the packages
and upgrade them, makes for a good starting point.

{% highlight bash %}
sudo apt-get update && sudo apt-get upgrade
{% endhighlight %}

### Install Oracle Java

Next, add the
[webupd8team](http://www.webupd8.org/2012/09/install-oracle-java-8-in-ubuntu-via-ppa.html)
oracle java repository so that it's easier to install the latest
Oracle Java and stay up to date. I plan to use Oracle Java as this is
recommended by Alfresco.

{% highlight bash %}
sudo add-apt-repository ppa:webupd8team/java
sudo apt-get update
{% endhighlight %}

Install Oracle Java first then set the environment variable JAVA_HOME
for your user (probably `vagrant`) *and* root since when you restart
services etc you'll need to use `sudo` which will look for JAVA_HOME
in root's environment variables. The best method? Set it for all users
in `/etc/environment`. However, Tomcat
[won't actually use your JAVA_HOME variable](http://askubuntu.com/questions/154953/specify-jdk-for-tomcat7)
so we symlink the java-8-oracle to default-java.

{% highlight bash %}
sudo apt-get -y install oracle-java8-installer oracle-java8-set-default
sudo update-java-alternatives -s java-8-oracle
sudo echo "JAVA_HOME=/usr/lib/jvm/java-8-oracle" | sudo tee -a /etc/environment
sudo ln -s /usr/lib/jvm/java-8-oracle /usr/lib/jvm/default-java
{% endhighlight %}


### Install Alfresco dependencies 

Alfresco has a few of dependencies, here is what they do:

+ Apache Tomcat - a JSP servlet web server, it takes HTTP requests from browser and serves back the application
+ PostgreSQL - a relational database, it stores your alfresco data
+ Postressql JDBC - a java database driver to access PostgreSQL
+ ImageMagick - a swiss-army knife that converts and manipulates images
+ Libreoffice - for converting between MS Office ans OASIS document formats

.. in addition, we install `unzip` so we can unzip the alfresco
community package. Not in Ubuntu by default. We install the
dependencies with the `-y` switch to answer 'yes' to all the questions
that apt-get throws up at you.

{% highlight bash %}
sudo apt-get -y install tomcat7 postgresql libpostgresql-jdbc-java imagemagick libreoffice unzip
{% endhighlight %}

### Get Alfresco 5 and move the web-server components into Tomcat

Get a copy of the Alfresco 5 Community addition, make a home for it
and unzip to it. I'm a fan of downloading install files to `/tmp` to
prevent polluting the home directory.

{% highlight bash %}
wget -O /tmp/alfresco-community-5.0.d.zip http://dl.alfresco.com/release/community/5.0.d-build-00002/alfresco-community-5.0.d.zip
sudo unzip /tmp/alfresco-community-5.0.d.zip
{% endhighlight %}

Next we install the WAR files from the alfresco zip into Tomcat, these
are basically
[JAR files for Web assets in Java lingo](https://en.wikipedia.org/wiki/WAR_(file_format)). I've
taken my steps from the
[Alfresco docs](http://docs.alfresco.com/community/tasks/alf-war-install.html)
so do look there for more information about what the files are and
what they do. Handily, the files you need are all in the web-server
directory in `/tmp/alfresco-community-5.0.d`, so we copy those across
and rename `/shared/classes/alfresco-global.properties.sample` to
`alfresco-global.properties` to set the global properties for
Alfresco.

{% highlight bash %}
sudo cp -R /tmp/alfresco-community-5.0.d/web-server/webapps/* /var/lib/tomcat7/webapps/
sudo cp -R /tmp/alfresco-community-5.0.d/web-server/shared/* /var/lib/tomcat7/shared/
sudo mv /var/lib/tomcat7/shared/classes/alfresco-global.properties.sample /var/lib/tomcat7/shared/classes/alfresco-global.properties
{% endhighlight %}

## Start Tomcat and marvel!

All that's left to do is restart tomcat and marvel at you lovely work
`http://localhost:9090/share/page/`. Ahhhhh.

