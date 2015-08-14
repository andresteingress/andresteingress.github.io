---
layout: post
title: Vagrant - Configuring a Solr Box
categories:
- vagrant
- solr
tags: []
status: publish
type: post
published: true
---
Recently I started some experiments with [Vagrant](http://www.vagrantup.com). Vagrant is a tool that lets you (pre-) configure development environments. This can be done
based on VirtualBox, Hyper-V, VMWare or many more so-called _providers_. You see, another way to put it is that Vagrant is a tool that helps
 with configuring, building and running virtual machine instances.

The default provider is VirtualBox but the documentation actually recommends to use VMWare for more stable and performant environments.

### Welcome to Vagrant

Vagrant can be installed via a manual download or your preferred package manager, such as `brew` on Mac OS.

We needed to write (I guess they're called) system tests, to test integration scenarios between an Apache Solr server
instance and a library. As you can imagine this implies that an Apache Solr instance needs to be installed on every developer
machine that intends to execute the entire test suite.

Now Vagrant comes into play. It's a command-line tool that is used to setup a virtual machine with all components pre-installed.

However, instead of configuring virtual machine images from scratch, Vagrant introduces the concept of _boxes_. Boxes are the package format that is
used to bring up identical development environments. The easiest way to use a box is to choose one from the [publicy available ones](https://vagrantcloud.com).

### The "precise64" Box

For our purpose, we based our Vagrant box on the "precise64" box. It contains an Ubuntu 12.04 LTS (precise) installation with some
tools pre-installed. The default box is specified in a file called `Vagrantfile`. 

Running `vagrant init` will create a template `Vagrantfile` in the current directory.

`Vagrantfile` is a file written in a Ruby DSL and it contains the configuration of our custom box:

</code></pre>ruby
Vagrant.configure("2") do |config|
  # Defines the Vagrant box name, download URL, IP and hostname
  config.vm.define :vagrant do |vagrant|
    vagrant.vm.box = "precise64"
    vagrant.vm.box_url = "http://files.vagrantup.com/precise64.box"

    vagrant.vm.network :private_network, ip: "192.168.66.6"
    vagrant.vm.network "forwarded_port", guest: 8983, host: 8898

    vagrant.vm.hostname = "vagrant.dcl"
  end
end
</code></pre>

The configuration specifies the pre-defined box `precise64` as "parent" vm box. In addition, it specifies the URL under which this box can be
 downloaded.

### More `Vagrantfile`

The `Vagrantfile` may consist of various sections. For a detailed overview of all available configuration options, please have a look at the [Vagrant documentation] (http://docs.vagrantup.com/v2/).

Next up in our configuration file is the network configuration. We use a private network,
this means we can access our guest from the host machine but the box won't be visible from the outside. It gets an IP address in the private
address space. Plus, we define a forwarded port. In fact, this is the port under which Apache Solr listens in the default configuration. Once
we access `localhost:8983` on the host machine, the request will be forwarded to the Vagrant virtual machine instance port 8983.

Would we start with `vagrant up` we would have a running Ubuntu 12.04 LTS instance within seconds. Unfortunately, Ubuntu 12.04 doesn't come with Solr pre-installed, so there's some work left for us.

As the pre-defined "precise64" box doesn't fix our use case, we need to alter the environment a bit. We need to install Java and Apache
Solr in our custom box. The process of adding/tweaking stuff in a box is called _provisioning_. The simplest way of implementing
provisioning is to write shell scripts. An advanced way would be to use tools such as [Chef](http://docs.vagrantup.com/v2/provisioning/chef_solo.html) or [Puppet](http://docs.vagrantup.com/v2/provisioning/puppet_apply.html).

We decided to use plain shell scripts and added a shell-script called `provision.sh` to our `Vagrantfile` configuration:

</code></pre>ruby
Vagrant.configure("2") do |config|

  config.vm.provision :shell, :inline => "
    sh /vagrant/scripts/provision.sh;
  "

  # Defines the Vagrant box name, download URL, IP and hostname
  config.vm.define :vagrant do |vagrant|
    vagrant.vm.box = "precise64"
    vagrant.vm.box_url = "http://files.vagrantup.com/precise64.box"

    vagrant.vm.network :private_network, ip: "192.168.66.6"
    vagrant.vm.network "forwarded_port", guest: 8983, host: 8983

    vagrant.vm.hostname = "vagrant.dcl"
  end
end
</code></pre>

The `provision.sh` shell-script defines all the commands to install Java and run the Solr instance:

</code></pre>bash
#! /bin/bash

##### VARIABLES #####

# Throughout this script, some variables are used, these are defined first.
# These variables can be altered to fit your specific needs or preferences.

# Server name
HOSTNAME="vagrant.dcl"

# Locale
LOCALE_LANGUAGE="en_US" # can be altered to your prefered locale, see http://docs.moodle.org/dev/Table_of_locales
LOCALE_CODESET="en_US.UTF-8"

# Timezone
TIMEZONE="Europe/Paris" # can be altered to your specific timezone, see http://manpages.ubuntu.com/manpages/jaunty/man3/DateTime::TimeZone::Catalog.3pm.html

VM_ID_ADDRESS="192.168.66.6"

#----- end of configurable variables -----#


##### PROVISION CHECK ######

# The provision check is intended to not run the full provision script when a box has already been provisioned.
# At the end of this script, a file is created on the vagrant box, we'll check if it exists now.
echo "[vagrant provisioning] Checking if the box was already provisioned..."

if [ -e "/home/vagrant/.provision_check" ]
then
  # Skipping provisioning if the box is already provisioned
  echo "[vagrant provisioning] The box is already provisioned..."
  exit
fi


##### PROVISION SOLR #####
echo "[vagrant provisioning] Updating mirrors in sources.list"

# prepend "mirror" entries to sources.list to let apt-get use the most performant mirror
sudo sed -i -e '1ideb mirror://mirrors.ubuntu.com/mirrors.txt precise main restricted universe multiverse\ndeb mirror://mirrors.ubuntu.com/mirrors.txt precise-updates main restricted universe multiverse\ndeb mirror://mirrors.ubuntu.com/mirrors.txt precise-backports main restricted universe multiverse\ndeb mirror://mirrors.ubuntu.com/mirrors.txt precise-security main restricted universe multiverse\n' /etc/apt/sources.list
sudo apt-get update

echo "[vagrant provisioning] Installing Java..."
sudo apt-get -y install curl
sudo apt-get -y install python-software-properties # adds add-apt-repository

sudo add-apt-repository -y ppa:webupd8team/java
sudo apt-get update

# automatic install of the Oracle JDK 7
echo oracle-java7-installer shared/accepted-oracle-license-v1-1 select true | sudo /usr/bin/debconf-set-selections

sudo apt-get -y install oracle-java7-set-default

export JAVA_HOME="/usr/lib/jvm/java-7-oracle/jre"

echo "[vagrant provisioning] Installing Apache Solr..."

sudo apt-get -y install unzip

curl -O http://tweedo.com/mirror/apache/lucene/solr/4.7.1/solr-4.7.1.zip
unzip solr-4.7.1.zip

rm solr-4.7.1.zip
cd solr-4.7.1/example/

# Solr startup
java -jar start.jar > /tmp/solr-server-log.txt &

echo "[vagrant provisioning] Bootstrapping Apache Solr..."

sleep 1
while ! grep -m1 'Registered new searcher' < /tmp/solr-server-log.txt; do
    sleep 1
done

echo "[vagrant provisioning] Apache Solr started. Index test data ..."

# Index some stuff
cd exampledocs/
java -jar post.jar solr.xml monitor.xml

##### CONFIGURATION #####

# Hostname
echo "[vagrant provisioning] Setting hostname..."
sudo hostname $HOSTNAME

##### CLEAN UP #####

sudo dpkg --configure -a # when upgrade or install doesn't run well (e.g. loss of connection) this may resolve quite a few issues
apt-get autoremove -y # remove obsolete packages

##### PROVISION CHECK #####

# Create .provision_check for the script to check on during a next vargant up.
echo "[vagrant provisioning] Creating .provision_check file..."
touch .provision_check
</code></pre>

The provisioning process starts with the definition of some variables and the check for the `.provision_check` file. This file will be touched
 once the provisioning process has gone through successfully. All files within the directory containing the `Vagrantfile` will be available
in the guest at `/vagrant`. The current home directory all file operations will be operated in during provisioning is `/home/vagrant`. Vagrant
allows to configure more of these so-called _synced folders_ but for our purposes the defaults were perfectly fine.

After the file check, `sources.list` will be updated and the following lines will be added at the beginning of the file:

</code></pre>bash
sudo sed -i -e '1ideb mirror://mirrors.ubuntu.com/mirrors.txt precise main restricted universe multiverse\ndeb mirror://mirrors.ubuntu.com/mirrors.txt precise-updates main restricted universe multiverse\ndeb mirror://mirrors.ubuntu.com/mirrors.txt precise-backports main restricted universe multiverse\ndeb mirror://mirrors.ubuntu.com/mirrors.txt precise-security main restricted universe multiverse\n' /etc/apt/sources.list
sudo apt-get update
</code></pre>

The `deb mirror:...` entries are used to optimize the selected apt-get mirror. This will result in the selection of the fastest available mirror
without any hard-coded local preliminaries.

The next lines are pretty straight-forward. The `ppa:webupd8team/java` repository is used to fetch Oracle JDK 7u51. As the JDK installer comes
with a modal confirmation dialog, the following lines are needed:

</code></pre>bash
echo oracle-java7-installer shared/accepted-oracle-license-v1-1 select true | sudo /usr/bin/debconf-set-selections
</code></pre>

This quietly confirms and installs the JDK automatically.

Next up is Apache Solr installation. The Solr 4.7.1 zip is downloaded and gets extracted in the home directory. Afterwards, the example
configuration is run with `java -jar start.jar`. This starts a Jetty instance. The script uses the following code to monitor when bootstrapping
is done:

</code></pre>bash
sleep 1
while ! grep -m1 'Registered new searcher' < /tmp/solr-server-log.txt; do
    sleep 1
done
</code></pre>

Once the message "Registered new searcher" appears we can safely assume the Solr instance is started.

Solr's `post.jar` can be used to add documents to the index. In this script we simply add the files from the `exampledocs` directory.

### Conclusion

And that's it.

With this configuration the development environment can be started via `vagrant up`. Once the machine shall be stopped, we can use
`vagrant halt` and `vagrant resume` to resume were we stopped.

What's really cool about it is that once we commit our `Vagrantfile` and `provision.sh` to our git repository every developer is free to check it out and run the `vagrant` command line tool with it. It results in exactly the same development virtual machine that can now be used to run our systems tests.


