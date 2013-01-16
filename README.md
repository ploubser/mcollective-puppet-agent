# MCollective Puppet Agent

This agent manages the *puppet agent*, unlike the older *puppetd* plugin
this one supports Puppet 3 and recent changes made to its locking and status files.

In addition to basic support for Puppet 3 this adds a number of new features, most them
usable under both Puppet 2.7 and 3.

  * Supports noop runs or no-noop runs
  * Supports limiting runs to certain tags
  * Support splay, no splay, splaylimits
  * Supports specifying a custom environment
  * Supports specifying a custom master host and port
  * Support Puppet 3 features like lock messages when disabling
  * Use the new summary plugins to provide convenient summaries where appropriate
  * Use the new validation plugins to provider richer input validation and better errors
  * Data sources for the current puppet agent status and the status of the most recent run

To use this agent you need:

  * MCollective 2.2.0 at least
  * Puppet 2.7 or 3.0

# TODO

  * Add a new puppet commander

## Agent Installation

Follow the basic [plugin install guide](http://projects.puppetlabs.com/projects/mcollective-plugins/wiki/InstalingPlugins)

## Configuring the agent

By default it just works but there are a few settings you can tweak in *server.cfg*:

    plugin.puppet.command=puppet agent
    plugin.puppet.splay=true
    plugin.puppet.splaylimit=30

These are the defaults, adjust to taste

## Usage
### Running Puppet

Most basic case is just a run:

    $ mco puppet runonce

...against a specific server and port:

    $ mco puppet runonce --server puppet.example.net:1234

...just some tags

    $ mco puppet runonce --tag one --tag two --tag three

...a noop run

    $ mco puppet runonce --noop

...a actual run when noop is set in the config file

    $ mco puppet runonce --no-noop

...in a specific environment

    $ mco puppet runonce --environment development

...a splay run

    $ mco puppet runonce --splay --splaylimit 120

...or if you have splay on by default and do not want to splay

    $ mco puppet runonce --no-splay

These can all be combined to your liking

### Requesting agent status

The status of the agent can be obtained:

    $ mco puppet status

     * [ ============================================================> ] 2 / 2

       dev1.example.net: Currently stopped; last completed run 9 minutes 11 seconds ago
       dev2.example.net: Currently stopped; last completed run 9 minutes 33 seconds ago

    Summary of Applying:

       false = 2

    Summary of Daemon Running:

       stopped = 2

    Summary of Enabled:

       enabled = 2

    Summary of Idling:

       false = 2

    Summary of Status:

       stopped = 2


    Finished processing 2 / 2 hosts in 45.01 ms

#### Requesting last run status

    $ mco puppet summary

     * [ ============================================================> ] 2 / 2


    Summary of Config Retrieval Time:

       Average: 0.13

    Summary of Total Resources:

       Average: 7

    Summary of Total Time:

       Average: 0.13


    Finished processing 2 / 2 hosts in 45.12 ms


#### Enabling and disabling

Puppet 3 supports a message when enabling and disableing

    $ mco rpc puppet disable message="doing some hand hacking"
    $ mco rpc puppet enable

The message will be displayed when requesting agent status if it is disabled,
when no message is supplied a default will be used that will include your
mcollective caller identity and the time

#### Running all enabled Puppet nodes

Often after committing a change you want the change to be rolled out to your
infrastructure as soon as possible within the performance constraints of your
infrastructure.

The performance of a Puppet Master generally comes down to the maximum concurrent
Puppet nodes that are applying a catalog it can sustain.

Using the MCollective infrastructure we can determine how many machines are
currently enabled and applying catalogs.

Thus to do a Puppet run of your entire infrastructure keeping the concurrent
Puppet runs as close as possible to 10 nodes at a time you would do:

    $ mco puppet runall 10

Below is the output from a run using a concurrency of 1 to highlight the output
you might expect:

    $ mco puppet runall 1
    2013-01-16 16:14:26: Running all nodes with a concurrency of 1
    2013-01-16 16:14:26: Discovering enabled Puppet nodes to manage
    2013-01-16 16:14:29: Found 2 enabled nodes
    2013-01-16 16:14:32: Currently 1 node applying the catalog; waiting for less than 1
    2013-01-16 16:14:37: dev1.example.net schedule status: Started a background Puppet run using the 'puppet agent --onetime --daemonize --color=false' command
    2013-01-16 16:14:40: Currently 1 node applying the catalog; waiting for less than 1
    2013-01-16 16:14:44: Currently 1 node applying the catalog; waiting for less than 1
    2013-01-16 16:14:48: Currently 1 node applying the catalog; waiting for less than 1
    2013-01-16 16:14:52: Currently 1 node applying the catalog; waiting for less than 1
    2013-01-16 16:14:56: Currently 1 node applying the catalog; waiting for less than 1
    2013-01-16 16:15:00: Currently 1 node applying the catalog; waiting for less than 1
    2013-01-16 16:15:04: Currently 1 node applying the catalog; waiting for less than 1
    2013-01-16 16:15:08: Currently 1 node applying the catalog; waiting for less than 1
    2013-01-16 16:15:13: dev2.example.net schedule status: Started a background Puppet run using the 'puppet agent --onetime --daemonize --color=false' command

Here we can see it first finds all machine that are enabled and then periodically
checks if the amount of instances currently applying a catalog is less than the
concurrency and then start as many machines as it can till it once again reaches
the concurrency limit.

#### Discovering based on agent status

Two data plugins are provided, to see what data is available about the running
agent do:

    $ mco rpc rpcutil get_data source=puppet
    Discovering hosts using the mc method for 2 second(s) .... 1

     * [ ============================================================> ] 1 / 1


    dev1.example.net
              applying: false
        daemon_present: false
       disable_message:
               enabled: true
               lastrun: 1348745262
         since_lastrun: 7776
                status: stopped

    Finished processing 1 / 1 hosts in 76.34 ms

You can then use any of this data in discovery, to restart apache on machines
with Puppet disable can now be done easily:

    $ mco rpc service restart service=httpd -S "puppet().enabled=false"

##### Discovery based on most recent run

You can restart apache on all machine that has File[/srv/www] managed by Puppet:

    $ mco rpc service restart service=httpd -S "resource('file[/srv/wwww]').managed=true"

...or machines that had many changes in the most recent run:

    $ mco rpc service restart service=httpd -S "resource().changed_resources>10"

...or that had failures

    $ mco rpc service restart service=httpd -S "resource().failed_resources>0"

Other available data include config_retrieval_time, config_version, lastrun,
out_of_sync_resources, since_lastrun, total_resources and total_time
