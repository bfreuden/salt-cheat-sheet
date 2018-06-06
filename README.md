
Salt documentation, getting started, and best practises:

https://docs.saltstack.com/en/latest/

https://docs.saltstack.com/en/getstarted/

https://docs.saltstack.com/en/latest/topics/best_practices.html

# Installation

Examples below are on Ubuntu 18.04.

Everything must (?) be run as root.

## Salt master

```shell 
root@saltmaster:~$ apt-get install salt-master
```

**WARNING**
Note that the salt-master install will create the */srv* directory (with the root:root owner it seems)
so chown it to salt:salt otherwise you will get silent errors when applying states!
```shell
root@saltmaster:~$ chown -R salt:salt /srv
```

## Salt minion

A minion is a machine you want to control.

```shell 
root@mediacenter:~$ apt-get install salt-minion
```
Minion configuration file is */etc/salt/minion*
You need to make 2 changes in it:
1. set the hostname of the salt master 
3. and set id of the minion

If the name of your master machine is *salt*, then this is the default and you don't need to do step 1.

```yaml
master: saltmaster
id: mediacenter
```

Having a DNS is certainly a good idea instead of having IP adresses hardcoded into files...

## Key configuration

See https://docs.saltstack.com/en/latest/ref/configuration/index.html

Get the public key of the master:
```shell 
root@saltmaster:~$ salt-key -F master
```

In */etc/salt/minion*, add the public key of the master (master.pub in the output of command above):
```yaml
master_finger: 'b1:d4:c2:a1:82:c7:04:08:56:14:a1:dd:cc:2e:4a:d0:f2:9b:7c:9a:36:5b:56:7c:f8:07:85:0d:7f:93:ae:d3'
```

Restart the minion:
```shell
root@mediacenter:~$ service salt-minion restart
```

Display the public key of the minion: 
```shell
root@mediacenter:~$ salt-call --local key.finger
```

Show all unaccepted keys on the master: 
```shell 
root@saltmaster:~$ salt-key -L
```

accept the minion key (if equal to the actual minion key, see above): 
```shell 
root@saltmaster:~$ salt-key -a mediacenter
```

or to accept all in one:
```shell 
root@saltmaster:~$ salt-key -A
```

test the connection (probably not testing the secure connection though): 
```shell 
salt '*' test.ping
```

# Remote command execution
See https://docs.saltstack.com/en/getstarted/fundamentals/remotex.html

Run commands on all machines:
```shell 
root@saltmaster:~$ salt '*' cmd.run 'ls /tmp'
```

show disk usage of all machines:
```shell 
root@saltmaster:~$ salt '*' disk.usage
```

install a package on all machines:
```shell 
root@saltmaster:~$ salt '*' pkg.install git
```

show network interfaces of all machines:
```shell 
root@saltmaster:~$ salt '*' network.interfaces
```

# Targetting
See https://docs.saltstack.com/en/getstarted/fundamentals/targeting.html

targetting specific machines:
```shell 
root@saltmaster:~$ salt 'media*' disk.usage
```

targetting using grains (grains are kind of properties of a salt host, either got from their own nature
like their OS, or manually added):
```shell 
root@saltmaster:~$ salt -G 'os:Ubuntu' test.ping
```

targetting using regex:
```shell 
root@saltmaster:~$ salt -E 'mediacenter[0-9]*' test.ping
```

targetting using a list:
```shell 
root@saltmaster:~$ salt -L 'mediacenter,masterminion' test.ping
```

targetting using a combination:
```shell 
root@saltmaster:~$ salt -C 'G@os:Ubuntu and *minion or S@192.168.50.*' test.ping
```

# Copying files

copying a ~/files.zip file (can large with --chunked) in the /tmp/ directory of minions:
```shell 
root@saltmaster:~$ salt-cp --chunked '*' ~/files.zip /tmp/
```

# Grains

See https://docs.saltstack.com/en/latest/topics/grains/

Grains are properties either got by Salt from the env, or manually set.

Listing grains:
```shell 
root@saltmaster:~$ salt '*' grains.ls
root@saltmaster:~$ salt '*' grains.items
```

# Pillars

(TDB)

# States

Salt state files are in YAML, so you probably want to have a look at this getting started: 

http://docs.ansible.com/ansible/latest/reference_appendices/YAMLSyntax.html

And you probably want to install a YAML validator:

```shell 
root@saltmaster:~$ apt-get install yamllint
```

### Creating a state file

States describe what is to be done or, more specifically, the expected state of the minion.

See https://docs.saltstack.com/en/latest/topics/tutorials/starting_states.html

create a file  /srv/salt/nettools.sls like this:
```yaml
---
install_network_packages:
  pkg.installed:
    - pkgs:
      - rsync
      - curl
...
```
Let's describe that file a little bit:
```yaml
---
# In theory (not mandatory though) a YAML file is starting with ---
# This 'install_network_packages' is the ID for a set of data, and it is called the ID Declaration.
# (It seems that it has to be unique if you have many of them).
# Sometimes that ID can be used by the module as a parameter.
install_network_packages:
  # Here we are using the 'pkg' state:
  # (see https://docs.saltstack.com/en/latest/ref/states/all/salt.states.pkg.html)
  # It is implemented by one of the pkg modules:
  # (see https://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.pkg.html)
  # Here you want packages to be installed (pkg.installed), but 
  # you could specify pkg.removed for instance.
  # You can also use the pkg state to add a custom repository.
  pkg.installed:
    # Then you have the list of "parameters" of the module
    # Here there is only a single item: the pkgs parameter
    - pkgs:
      # That parameter is accepting a list of packages to be installed
      - rsync
      - curl
# In theory (not mandatory though) a YAML file is ending with ...
...
```

validate it with: 
```shell 
root@saltmaster:~$ yamllint nettools.sls 
```

Apply it on a minion:
```shell 
root@saltmaster:~$ salt 'mediacenter' state.apply nettools
```

Calling state.apply with no arguments starts a highstate (global state application?).
Either on all minions:
```shell
root@saltmaster:~$ salt '*' state.apply
```
Or on specific ones:
```shell 
root@saltmaster:~$ salt 'mediacenter' state.apply
```

## The top file

In Salt, the file which contains a mapping between groups of machines on a network and the configuration roles that should be applied to them is called a *top file*.

See https://docs.saltstack.com/en/latest/ref/states/top.html

There is one top file per environment (base, dev, prod) for instance. The 'base' environment is defined in Salt's default configuration. Those environments are either independent, or merged (see the */etc/salt/master* config).

The top.sls file has to be located by default in */srv/salt/top.sls*

The top file is used to define which state has to be applied on which minions:
```yaml
---
base:
  '*':
    - nettools
...
```
Let's describe that file a little bit:
```yaml
---
# This is the top file of the 'base' environment
base:
  # On all minions
  '*':
    # We want to apply the configuration defined in the nettools.sls file
    - nettools
...
```


You can locally run your state (if the config files are on the minion, which is generally not the case).

Let's assume with have a minion of the saltmaster machine too. It is very convenient to test the configuration.
You can show the state of a minion (--local is a debug feature that makes the minion read state files locally instead of asking them to the master):
```shell 
root@saltmaster:~$ salt-call --local state.show_highstate
```

You can report what would be done if the state "executed" on the minion (dry run):
```shell 
root@saltmaster:~$ salt-call --local state.highstate test=True
```
Not sure those two commands would work on a machine without salt configuration files in /srv.

ask the minion its "vision" of top:
```shell 
root@saltmaster:~$ salt 'minion' state.show_top
```

get a report of dependency versions:
```shell 
root@saltmaster:~$ salt-call --versions-report
```

# Jinja temlates

(TBD)

See http://jinja.pocoo.org/docs/2.10/templates/

validating a jinja file can be done using the slsutil.renderer module:
```shell 
salt 'masterminion' slsutil.renderer /srv/salt/users.sls 'jinja'
```
or:
```shell 
salt-call --local  slsutil.renderer /srv/salt/users.sls 'jinja'
```

# Salt ssh

Salt can be used in agentless mode, without minion. 

(TBD)

# Salt cloud

Salt can be used to manage (provisioning, starting, stopping) cloud instances (Amazon, Azure...)

(TBD)


