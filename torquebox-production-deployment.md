# TorqueBox in Production - Basic Deployment

Let me start with a caveat - this is a fairly trivial deployment. Things can easily get more complex if you are trying to get a cluster up and running on AWS, for example. But this is a good place to start if you'd like to get TorqueBox up and running in a production environment.  Examples in this article are based on a real world system that will soon be public.  For tips and tricks on getting your development environment going, check out blog posts on [RVM] and [torquebox-server]. There's also a decent 5 minute screencast on getting TorqueBox going on your development system.

## Server Specs

For this example, the entire application is running on a single 1024 Linode. That's a gig of RAM, 40 gigs of disk, and 400GB data transfer and it will be running TorqueBox, PostgreSQL, Apache, and a Rails application. The operating system is a Fedora 15 stock install. Additional services like PostgreSQL and Apache were installed with `yum install ...`.  Getting the server up and running was basically point and click on the Linode website as it should be for just about any VPS provider.  

Here are the packages I installed on top of the stock distribution.

- `postgresql-server` for the application database
- `java-1.6.0-openjdk` for the Java runtime
- `git` for application deployment via SCM
- `httpd` for the Apache web server
- `mod_cluster` for request dispatching 

Adjust the list as necessary for your application. We'll cover Apache and `mod_cluster` setup below.

## TorqueBox Installation

I love RVM and use it daily for development. But I've always felt a little uncomfortable about it in production. And if the talk in #torquebox is any indication, my reservations were not misguided. It can be a headache and you should avoid it if possible.  Here we're using the TorqueBox binary download. With this you get TorqueBox+JBoss and JRuby 1.6.7 (as of this writing) already primed with all of the torquebox* gems.  Download it all and unzip it in `/opt/torquebox`. Here's what it looks like with output removed for brevity. 

This assumes you've already created a `torquebox` user. 

    $ wget http://torquebox.org/release/org/torquebox/torquebox-dist/2.0.0.cr1/torquebox-dist-2.0.0.cr1-bin.zip
    $ mkdir /opt/torquebox
    $ chown torquebox:torquebox /opt/torquebox
    $ su torquebox
    $ unzip torquebox-dist-2.0.0.cr1-bin.zip -d /opt/torquebox/
    $ cd /opt/torquebox
    $ ln -s torquebox-dist-2.0.0.cr1 current
    
Now make sure that all users have access to the TorqueBox environment, and that the PATH is adjusted accordingly. For Fedora, this means adding a script to `/etc/profile.d`.

    $ cat > /etc/profile.d/torquebox.sh
       export TORQUEBOX_HOME=/opt/torquebox/current
       export JBOSS_HOME=$TORQUEBOX_HOME/jboss
       export JRUBY_HOME=$TORQUEBOX_HOME/jruby
       PATH=$JBOSS_HOME/bin:$JRUBY_HOME/bin:$PATH
       ^D
       
Login with a new shell and see if everything works as it should.

    [root@torquebox ~]# torquebox
    Tasks:
      torquebox deploy ROOT        # Deploy an application to TorqueBox
      torquebox undeploy ROOT      # Undeploy an application from TorqueBox
      torquebox run                # Run TorqueBox
      torquebox rails ROOT         # Create a Rails application at ROOT using the...
      torquebox archive ROOT       # Create a nice self-contained application arc...
      torquebox cli                # Run the JBoss AS7 CLI
      torquebox env [VARIABLE]     # Display TorqueBox environment variables
      torquebox help [TASK]        # Describe available tasks or one specific task
      torquebox list applications  # List applications deployed to TorqueBox and ...

You might even want to start the server once without any applications deployed just to make sure that things have gone smoothly so far.  Just type `torquebox run`. This is kind of like `./script/server` in a Rails application. You should see the server starting itself up; lots of log messages - but no errors if everything is working as it should.  The last two lines should look something like this:

    10:36:33,446 INFO  [org.torquebox.core.runtime] (MSC service thread 1-6) Created ruby runtime (ruby_version: RUBY1_8, compile_mode: JIT, context: global) in 4.48s
    10:36:33,488 INFO  [org.jboss.as] (Controller Boot Thread) JBAS015874: JBoss AS 7.1.0.Final "Thunder" started in 14335ms - Started 170 of 259 services (88 services are passive or on-demand)
       
You can just type `^C` to kill the server and continue to set up your system.


## TorqueBox as a Startup Service

Since we're talking about a production system, you'll want TorqueBox to run as a service and startup when your server starts. Fedora 15 has shifted from Upstart to systemd, and while TorqueBox comes with some upstart friendlyness, we have yet to add [systemd suppprt].  Instead, you'll need to handle it manually.  To do this, we can make use of some bits that come with JBoss (and therefore TorqueBox) out of the box.

    $ cp $JBOSS_HOME/bin/init.d/jboss-as-standalone.sh /etc/init.d/jboss-as-standalone
    
The `jboss-as-standalone` startup script makes use of a few environment variables that can be set by creating a `jboss-as.conf` file in `/etc/jboss-as`. But the version of this configuration file that ships with JBoss needs some changes for the system we've setup so far.  Here's what yours should look like.

    [root@torquebox etc]# cat /etc/jboss-as/jboss-as.conf 
    # General configuration for the init.d script
    JBOSS_USER=torquebox
    JBOSS_HOME=/opt/torquebox/current/jboss
    JBOSS_PIDFILE=/var/run/torquebox/torquebox.pid
    JBOSS_CONSOLE_LOG=/var/log/torquebox/console.log
    JBOSS_CONFIG=standalone-ha.xml
    
Now, if everything is done correctly, you can check your installation by running `service jboss-as-standalone start`. Works OK? Great - let's move on.

NB: I believe there is more to do here in order to ensure that the system starts jboss, but my sysadmin skillz are rusty. Comments welcome.

## Supporting Servers and Software

As with MRI, a TorqueBox production box will typically have a request dispatcher fronting the application, accepting web requests and handing them off to your app.  In this case, we're going to use Apache and `mod_cluster` to achieve that. Even though we're not running a cluster of servers, `mod_cluster` makes it very simple to get Apache and TorqueBox talking with each other. And when the application does outgrow a single backend, it's trivial to add more to the cluster.

### `mod_cluster` configuration

## Deploying with Capistrano
