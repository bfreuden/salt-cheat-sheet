
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

# Pillars

See https://docs.saltstack.com/en/latest/topics/pillar/

Pillar is an interface for Salt designed to offer global values that can be distributed to minions. 

Pillars are not containing states, and they are not containing a description of what is to be done. Pillars are only containing data that will be used as an input for writing states.

A pillar file is also an sls file written in YAML, and its *schema* (its structure) is up to you.

By default, for the 'base' environment they are stored in the */srv/pillar* directory.

For example here is a */srv/pillar/users.sls* pillar file describing users:
```yaml 
---
users:
  bob:
    fullname: Bob
    password: $6$axWX[...]/fjtG1
    groups:
      - users
    ssh_auth:
      - ssh-rsa AAAAB3NzaC1yc2EA[...]bcgnkzrKn/WkgfJfViYLw==
    user_files:
      enabled: true
  alice:
    fullname: Alice
    password: $6$xr65i[...]6o7JWes1
    groups:
      - sudo
      - users
    user_files:
      enabled: true
...
```
The structure of this file is really up to you.

Then you must have a top.sls file to reference it. For instance here is a */srv/pillar/top.sls* file that will state that this user configuration is for all machines:


```yaml 
---
base:
  '*':
    - users
...
```

# Jinja temlates

See http://jinja.pocoo.org/docs/2.10/templates/

Jinja is a template engine that is able to generate any text file using variables, expressions and directives.

A Jinja template contains variables and/or expressions, which get replaced with values when a template is rendered; and tags, which control the logic of the template. The template syntax is heavily inspired by Django and Python.

Within Salt, the Jinja engine will operate on grain and/or pillar data.

## Jinja pillars

Here is a */srv/pillar/pkgs.sls* pillar that is defining the name of packages depending on the grains data:

```HTML+Django 
---
pkgs:
	{% if grains['os'] == 'RedHat' %}
	apache: httpd
	git: git
	{% elif grains['os'] == 'Debian' %}
	apache: apache2
	git: git-core
	{% endif %}
...
```
That's an illustration of Jinja if statement:

http://jinja.pocoo.org/docs/2.10/templates/#if


## Validating Jinjas

Validating a jinja file can be done using the slsutil.renderer module:
```shell 
salt 'masterminion' slsutil.renderer /srv/salt/users.sls 'jinja'
```
or:
```shell 
salt-call --local  slsutil.renderer /srv/salt/users.sls 'jinja'
```

Note that those command do not tell you if the *execution* of your file will succeed on minions, they will just tell you if there is a Jinja syntax error.


## Jinja states

Let's start with a simple */srv/salt/pkgs.sls* state that is based on the */srv/pillar/pkgs.sls* pillar.

That example is an illustration of Jinja variables: http://jinja.pocoo.org/docs/2.10/templates/#variables

```yaml
---
apache:
  pkg.installed:
    - name: {{ pillar['pkgs']['apache'] }}
...
```

You can also create more complex states like this */srv/salt/users.sls* state based on the */srv/pillar/users.sls* we defined above.

That example is an illustration of Jinja for control structure: http://jinja.pocoo.org/docs/2.10/templates/#for

Let's start with a non-commented version, then we'll add comments:

```HTML+Django 
---
{% for name, user in pillar.get('users', {}).items() if user.absent is not defined or not user.absent %}
{%- if user == None -%}
{%- set user = {} -%}
{%- endif -%}
{%- set user_files = salt['pillar.get'](('users:' ~ name ~ ':user_files'), {'enabled': False}) -%}
{%- set home = user.get('home', "/home/%s" %name) -%}
{%- set user_group = name -%}
{% for group in user.get('groups', []) %}
users_{{name}}_{{group}}_group:
  group:
    - name: {{group}}
    - present
{% endfor %}
users_{{name}}_user:
  group.present:
    - name: {{ user_group }}
    - gid: {{ user['uid'] }}
  user.present:
    - name: {{ name }}
    - home: {{ home }}
    - uid: {{ user['uid'] }}
    - password: {{ user['password'] }}
    - fullname: {{ user['fullname'] }}
    - groups:
        - {{ user_group }}
        {% for group in user.get('groups', []) %}
        - {{ group }}
        {% endfor %}
{% if 'ssh_auth' in user %}
{% for auth in user['ssh_auth'] %}
users_ssh_auth_{{name}}_{{loop.index0 }}:
  ssh_auth.present:
    - user: {{ name }}
    - name: {{ auth }}
{% endfor %}
{% endif %}
{% if user_files.enabled %}
vimrc_{{name}}:
  file.managed:
    - name: {{home}}/.vimrc
    - source: salt://files/.vimrc
{% endif %}
{% endfor %}
...
```

Here is the commented out version:

```HTML+Django 
---
# This is a Jinja template file (see http://jinja.pocoo.org/docs/2.10/templates/)
# loop on dictionary that is below the 'users' key of the pillar
# and use an empty dictionary {} if no user specified below users
# then put the key of the current entry in the 'name' variable, 
# and put the value of the current entry (a dictionary) in the 'user' variable.
# maybe that code also allows to disable a user with the user.absent stuff?
{% for name, user in pillar.get('users', {}).items() if user.absent is not defined or not user.absent %}

# if the user dictionary is not defined, use an empty dictionary
# the minus - sign means that this template line will not generate an empty line
# (see "Whitespace Control" in http://jinja.pocoo.org/docs/2.10/templates/)
{%- if user == None -%}
{%- set user = {} -%}
{%- endif -%}

# here we are doing several dictionary lookups (users -> name -> user_files) to get the value of that 
# user_files dictionary entry. If it is not defined, we will be using the {'enabled': False} dictionary.
{%- set user_files = salt['pillar.get'](('users:' ~ name ~ ':user_files'), {'enabled': False}) -%}

# here we are getting the value of the home key in the user dictionary, or building it from the
# user name if it does not exist
{%- set home = user.get('home', "/home/%s" %name) -%}

# set the user_group variable to be the name of the user
{%- set user_group = name -%}

# at this point, we have only read pillar data, and set Jinja variables
# now we will write salt states

# first we loop on groups of the users (defined in the pillar)
{% for group in user.get('groups', []) %}
# and we are creating a salt state using the group module of salt
# https://docs.saltstack.com/en/2017.7/ref/modules/all/salt.modules.groupadd.html#module-salt.modules.groupadd
# it looks like salt state name has to be unique, and we make sure it is
# by using the user name and the group and a bit of context around
# so here we are declaring that each group of the user (declared in the pillar) must be present on the machine
users_{{name}}_{{group}}_group:
  group:
    - name: {{group}}
    - present
{% endfor %}

# then create the user
users_{{name}}_user:
  # maybe this is a trick to make sure the group is present before creating the user?
  # actually I don't know the difference between "group - present" above and "group.present" below:
  group.present:
    - name: {{ user_group }}
    - gid: {{ user['uid'] }}
  # now we are making sure the user is present
  # otherwise it will be created with the following properties
  user.present:
    - name: {{ name }}
    - home: {{ home }}
    - uid: {{ user['uid'] }}
    - password: {{ user['password'] }}
    - fullname: {{ user['fullname'] }}
    - groups:
        - {{ user_group }}
        # again a loop on users groups
        {% for group in user.get('groups', []) %}
        - {{ group }}
        {% endfor %}

# now setup our public ssh keys
{% if 'ssh_auth' in user %}
{% for auth in user['ssh_auth'] %}
# again, here we are trying hard to have unique names for salt states by using the loop index:
users_ssh_auth_{{name}}_{{loop.index0 }}:
  # here we are using the salt ssh_auth module 
  # (see https://docs.saltstack.com/en/latest/ref/states/all/salt.states.ssh_auth.html)
  # to add our public keys to the users
  ssh_auth.present:
    - user: {{ name }}
    - name: {{ auth }}
{% endfor %}
{% endif %}

# now we are going to copy some files in the home directory
{% if user_files.enabled %}
vimrc_{{name}}:
  # we will do so using the file module of salt
  # https://docs.saltstack.com/en/develop/ref/modules/all/salt.modules.file.html
  file.managed:
    - name: {{home}}/.vimrc
    - source: salt://files/.vimrc
{% endif %}

# end loop on users
{% endfor %}
...
```


# Salt ssh

Salt can be used in agentless mode, without minion. 

(TBD)

# Salt cloud

Salt can be used to manage (provisioning, starting, stopping) cloud instances (Amazon, Azure...)

(TBD)


