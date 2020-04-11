
Salt documentation, getting started, and best practises:

https://docs.saltstack.com/en/latest/

https://docs.saltstack.com/en/getstarted/

https://docs.saltstack.com/en/latest/topics/best_practices.html

# Installation

Examples below are on Ubuntu 18.04.

Everything must (?) be run as root.

```bash
sudo -s
```

## Salt master

```bash
root@saltmaster:~$ apt-get install salt-master
```

**WARNING**
Note that the salt-master install will create the */srv* directory (with the root:root owner it seems)
so chown it to salt:salt otherwise you will get silent errors when applying states!
```bash
root@saltmaster:~$ chown -R salt:salt /srv
```

## Salt minion

A minion is a machine you want to control.

```bash
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
```bash
root@saltmaster:~$ salt-key -F master
```

In */etc/salt/minion*, add the public key of the master (master.pub in the output of command above):
```yaml
master_finger: 'b1:d4:c2:a1:82:c7:04:08:56:14:a1:dd:cc:2e:4a:d0:f2:9b:7c:9a:36:5b:56:7c:f8:07:85:0d:7f:93:ae:d3'
```

Restart the minion:
```bash
root@mediacenter:~$ service salt-minion restart
```

Display the public key of the minion:
```bash
root@mediacenter:~$ salt-call --local key.finger
```

Show all unaccepted keys on the master:
```bash
root@saltmaster:~$ salt-key -L
```

accept the minion key (if equal to the actual minion key, see above):
```bash
root@saltmaster:~$ salt-key -a mediacenter
```

or to accept all in one:
```bash
root@saltmaster:~$ salt-key -A
```

test the connection (probably not testing the secure connection though):
```bash
salt '*' test.ping
```

# Remote command execution
See https://docs.saltstack.com/en/getstarted/fundamentals/remotex.html

Run commands on all machines:
```bash
root@saltmaster:~$ salt '*' cmd.run 'ls /tmp'
```

show disk usage of all machines:
```bash
root@saltmaster:~$ salt '*' disk.usage
```

install a package on all machines:
```bash
root@saltmaster:~$ salt '*' pkg.install git
```

show network interfaces of all machines:
```bash
root@saltmaster:~$ salt '*' network.interfaces
```

# Targetting
See https://docs.saltstack.com/en/getstarted/fundamentals/targeting.html

targetting specific machines:
```bash
root@saltmaster:~$ salt 'media*' disk.usage
```

targetting using grains (grains are kind of properties of a salt host, either got from their own nature
like their OS, or manually added):
```bash
root@saltmaster:~$ salt -G 'os:Ubuntu' test.ping
```

targetting using regex:
```bash
root@saltmaster:~$ salt -E 'mediacenter[0-9]*' test.ping
```

targetting using a list:
```bash
root@saltmaster:~$ salt -L 'mediacenter,masterminion' test.ping
```

targetting using a combination:
```bash
root@saltmaster:~$ salt -C 'G@os:Ubuntu and *minion or S@192.168.50.*' test.ping
```

# Copying files

copying a ~/files.zip file (can large with --chunked) in the /tmp/ directory of minions:
```bash
root@saltmaster:~$ salt-cp --chunked '*' ~/files.zip /tmp/
```

# Grains

See https://docs.saltstack.com/en/latest/topics/grains/

Grains are properties either got by Salt from the env, or manually set.

Listing grains:
```bash
root@saltmaster:~$ salt '*' grains.ls
root@saltmaster:~$ salt '*' grains.items
```

# States

Salt state files are in YAML, so you probably want to have a look at this getting started:

http://docs.ansible.com/ansible/latest/reference_appendices/YAMLSyntax.html

And you probably want to install a YAML validator:

```bash
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
```bash
root@saltmaster:~$ yamllint nettools.sls
```

Apply it on a minion:
```bash
root@saltmaster:~$ salt 'mediacenter' state.apply nettools
```

Calling state.apply with no arguments starts a highstate (global state application?).
Either on all minions:
```bash
root@saltmaster:~$ salt '*' state.apply
```
Or on specific ones:
```bash
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
```bash
root@saltmaster:~$ salt-call --local state.show_highstate
```

You can report what would be done if the state "executed" on the minion (dry run):
```bash
root@saltmaster:~$ salt-call --local state.highstate test=True
```

Of course on other hosts, the minion would not be able to access new files locally, so:
```bash
root@saltminion:~# salt-call state.highstate test=True
```

That test on the mediacenter host can also be run from the salt master (where you're developing):
```bash
salt 'saltminion' state.apply test=True
```

Ask the local minion its "vision" of top:
```bash
root@saltmaster:~$ salt 'minion' state.show_top
```

Get a report of dependency versions:
```bash
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

WARNING: please note that pillar Jinja can only work from grain data.
AFAIK you cannot write a a pillar Jinja (let's say samba user configuration) based on another pillar (listing your users).
This limitation is described here:

https://github.com/saltstack/salt/issues/6955

It seems that Saltclass can overcome this limitation:

https://docs.saltstack.com/en/latest/ref/tops/all/salt.tops.saltclass.html#module-salt.tops.saltclass

There is some example data here:

https://git.mauras.ch/salt/saltclass/src/branch/master/examples


## Validating Jinjas

Validating a jinja file can be done using the slsutil.renderer module:
```bash
salt 'masterminion' slsutil.renderer /srv/salt/users.sls 'jinja'
```
or:
```bash
salt-call --local  slsutil.renderer /srv/salt/users.sls 'jinja'
```

Note that those command do not tell you if the *execution* of your file will succeed on minions, they will just tell you if there is a Jinja syntax error.

WARNING: pay a great attention to Jinja *Whitespace Control* (described in http://jinja.pocoo.org/docs/2.10/templates/).

The following Jinja will generate an empty line after the users, and YAML will complain:
```HTML+Django
---
  users:
{% for name, user in pillar.get('users', {}).items() %}
```

This one (notice the - character after {%) will be ok since no empty line will be generated:
```HTML+Django
---
  users:
{%- for name, user in pillar.get('users', {}).items() -%}
```

Maybe this one would work too (notice indentation before {%) but I've not tested:
```HTML+Django
---
  users:
    {% for name, user in pillar.get('users', {}).items() %}
```

## Jinja states

Let's start with a simple */srv/salt/pkgs.sls* state that is based on the */srv/pillar/pkgs.sls* pillar.

This example is an illustration of Jinja variables: http://jinja.pocoo.org/docs/2.10/templates/#variables

```yaml
---
apache:
  pkg.installed:
    - name: {{ pillar['pkgs']['apache'] }}
...
```

You can also create more complex states like this */srv/salt/users.sls* state based on the */srv/pillar/users.sls* we defined above.

This example is an illustration of Jinja for control structure: http://jinja.pocoo.org/docs/2.10/templates/#for

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

```yaml
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


# Salt formulas

There are many salt states already written by the salt community, serving many purposes.
Those are called salt formulas (see https://docs.saltstack.com/en/latest/topics/development/conventions/formulas.html)

You can find plenty of them on this github (apache, php, samba...): https://github.com/saltstack-formulas

Altough you can clone or download formula files by yourself, salt can do that for you through its GitFS fileserver backend. You can directly use formulas stored in the saltstack-formulas Github account, or you can fork them on your own account for more stability (your version won't be updated).

In */etc/salt/master* add those lines (they are commented-out somewhere in the file) to use the samba formula:
```yaml
fileserver_backend:
  - roots
  - gitfs

gitfs_remotes:
  - git://github.com/bfreuden/samba-formula.git
```

Then restart the salt master:
```bash
root@saltmaster:~$ service salt-master restart
```

Each Github formula repository has a directory containing state files: for instance the *samba* directory for the samba-formula repository (https://github.com/bfreuden/samba-formula/tree/master/samba)

When formulas are configurable, they also have an *pillar.example* file next to it (https://github.com/bfreuden/samba-formula/blob/master/pillar.example)

To use the formula in your state files, you have to mention this directory name in your */srv/salt/top.sls* file:
```yaml
---
base:
  'mediacenter':
    - samba
    - samba.config
...
```

To override the default configuration, we must now define a */srv/pillar/samba.sls* based on the *pillar.example* file.
Something like this should be enough:
```yaml
samba:
  conf:
    render:
      section_order: ['global', 'user1share']
    sections:
      global:
        workgroup: workgroup
      user1share:
        comment: "user1 samba share"
        path: /home/user1
        valid users: user1
        create mode: '0660'
        directory mode: '0770'
        public: no
        writable: yes
        printable: no

  users:
    user1:
      password: user1sambapassword
    user2:
      password: user2sambapassword
```

I am generally not editing my salt files on place and editing file in my home directory,
then copying them into /srv (for many reasons, but mostly because of the salt:salt owner of the files).

So let's rsync files from my home directory to /srv and change the owner:
```bash
root@saltmaster:~/saltdev/srv# rsync -avz --exclude '.git' /home/bruno/saltdev/srv/ /srv/ ; chown -R salt:salt /srv
```

Now let's test that before actually deploying it:
```bash
root@saltmaster:~/saltdev/srv/pillar# salt 'mediacenter' state.apply test=True
```

If you have used Jinja you can validate your pillar with:
```bash
root@saltmaster:~/saltdev/srv/pillar# salt-call --local  slsutil.renderer /srv/pillar/samba.sls 'jinja'
```

# Storing sensitive data (passwords)

Salt can leverage GPG encryption inside sls files (https://docs.saltstack.com/en/latest/ref/renderers/all/salt.renderers.gpg.html).

This guy has been incredibly helpful:
https://gist.github.com/vrillusions/5484422

This is how to set this up (creates a *saltmaster* key):
```bash
root@saltmaster:~/saltdev/srv/pillar#
mkdir -p /etc/salt/gpgkeys
chmod 0700 /etc/salt/gpgkeys
cat <<EOT >> /tmp/genkey-batch
%no-protection
Key-Type: default
Subkey-Type: default
Name-Real: saltmaster
Name-Email: saltmaster@saltstack.com
Expire-Date: 0
EOT
export GNUPGHOME=/etc/salt/gpgkeys
gpg --batch --gen-key /tmp/genkey-batch
```

Then export the *saltmaster* public key:
```bash
root@saltmaster:~/saltdev/srv/pillar#
gpg --homedir /etc/salt/gpgkeys --armor --export saltmaster > /tmp/saltmaster_pubkey.gpg
```

Then import the *saltmaster* public key into your own keyring:
```bash
me@saltmaster:~/saltdev$ gpg --import /tmp/saltmaster_pubkey.gpg
```
Now you can use the *saltmaster* public key to encrypt data:
```bash
me@saltmaster:~/saltdev$ echo -n "supersecret" | gpg --armor --batch --trust-model always --encrypt -r saltmaster
```

The encrypted data can be put into your yaml files. Just make sure the file is starting
with *#!yaml|gpg* and use the following syntax (pay attention to the indentation! the file CANNOT start with --- like the others!):
```yaml
#!yaml|gpg
users:
  bruno:
    fullname: Me
    samba:
      password: |
        -----BEGIN PGP MESSAGE-----

        hQGMAzAL2NXA7O5UAQv/XKcFQHs7AovUR8T/c4AwQBjHVNMrPGIHQ7N3ZN22m2kP
        zqbJnn13SdOUlxW9u76UvP/g2pf5W/hUWyri3gPGCF65s92rHLsRcuAvVNugWQNH
        IOywRt2G5PhUdSFC6xr6OnVivOglCnReCmjJgWimLGBfetnePUzpLF4CtdTZGF2g
        lcGv/GyaWCk86s/cF15rMr2s+9tMKcD/2YbdoOEgUTgt0gxuAspz8yvb1TMH4a92
        V5s/Ve68KjHxNCutNgdBnH33NXlEymL+hdD4mLKDHxNQCrwuAazrj65QqFkmiF/P
        ZdvRfwfhYwuAIUDQArwI4s4QIPJ4iQ4Opo1J8f0OlykHp4klRApQa3NzCOuLecFG
        1B1HBM/hD2hedOuDpjt6XWLhzqTMOdMF3BNyrNlYK6OLFis70kjdWEKcGMIqNiTA
        FjAFq7c0tv+XjYhNcT9re2IIoBWSM3yTWbncA2kcHfmcun6h3X76OXoZez+g+65g
        Wf0utBVEZHQJd/kF5yYu0kMBOcAwEOhpndv9iW5cax+mT1UGo9d/jxI+sURhX0t+
        9/LZp5enwIy1khfXxs9vv4EmBkyADkT6/8eNqzhEXjA8uRQz
        =kzuB
        -----END PGP MESSAGE-----
...
```

# Salt ssh

Salt can be used in agentless mode, without minion.

https://docs.saltstack.com/en/latest/topics/ssh/

One probably has to think twice before using salt-ssh because Ansible might be a best contender here...

Salt is an agent-based solution by design: salt master is a server, salt minions are servers.
When minions are applying a formula requiring files, they can simply download them from the server.

With salt-ssh when a file is required by a formula, it must be sent in along with the formula. 
Analyzing formulas to find files is apparently a difficult task when Jinja is involved. 
So salt-ssh has an --extra-filerefs option to manually specify a file to be included. 
But doing so places the burder on the user of the formula (who doesn't know it!).

On top of that --extra-filerefs doesn't seem to work well when targeting a file on gitfs.
So there are times you can't simply use gitfs. So you have to clone the formula and declare it file roots
of your salt environment.

see: https://github.com/saltstack/salt/issues/19564

## Installation (as non root user)
The following setup is a bit complex but it seems it is required to be able to run salt-ssh in non root.

First install salt-ssh:
```bash
sudo apt-get install salt-ssh
```

Then create salt_setup (the name is up to you) in your home directory:
```bash
cd
mkdir salt_setup
cd salt_setup
mkdir -p {config,salt/{files,templates,states,pillar,formulas,pki/master,logs}}
mkdir cache
touch ssh.log
cd
```

Then copy the content of /etc/salt here:
```bash
cp -rp /etc/salt/* ~/salt_setup/
chown $USER. -R ~/salt_setup
```

Finally create master and Saltfile files:
```bash
cat <<EOT >> ~/salt_setup/master
root_dir: "/home/$USER/salt_setup"
pki_dir: "pki"
cachedir: "cache"
log_file: "salt-ssh.log"

file_roots:
  base:
    - /home/$USER/salt_setup/salt

fileserver_backend:
  - roots
  - gitfs
EOT

cat <<EOT >> ~/salt_setup/Saltfile
salt-ssh:
  config_dir: "/home/$USER/salt_setup/"
  log_file: "/home/$USER/salt_setup/ssh.log"
  pki_dir: "/home/$USER/salt_setup/pki"
  cachedir: "/home/$USER/salt_setup/cache"
  roster_file: "/home/$USER/salt_setup/roster"
#  ssh_wipe: True
EOT
```

## Configuring managed servers

Declare servers in roster file:
```bash
for server in server1 server2 server3; do \
echo "$server:" >> ~/salt_setup/roster ; \
echo "  host: $server" >> ~/salt_setup/roster ; \
echo "  user: $USER" >> ~/salt_setup/roster ; \
echo "  sudo: true" >> ~/salt_setup/roster ; \
echo "" >> ~/salt_setup/roster ; \
done
```

It must give something like this:
/etc/salt/roster
```yaml
server1:
  host: server1
  user: me
  sudo: true

server2:
  host: server2
  user: me
  sudo: true

server3:
  host: server3
  user: me
  sudo: true
```

Please note that the procedure below is not the recommended way of installing keys and salt-ssh can do that for you, see:
https://youtu.be/qWG5pI8Glbs

Salt-ssh can also ask for your password if you prefer not deploying your key.

Add SSH key on your freshly installed machines:
```bash
for server in server1 server2 server3; do \
ssh $USER@$server "mkdir .ssh ; chmod 700 .ssh ; echo \"`cat ~/.ssh/id_rsa.pub`\" >> .ssh/authorized_keys" ; \
done
```

Then enable sudo NOPASSWD for your user on all machines 
```bash
for server in server1 server2 server3; do \
ssh -t $USER@$server "echo '$USER ALL=(ALL) NOPASSWD: ALL' | sudo tee -a /etc/sudoers" ; \
done
```

## Running salt-ssh

Note: to run salt-ssh as non-root you must be in the ~/salt_setup directory.
```bash
cd ~/salt_setup
```

Finally run your first salt-ssh command:
```bash
salt-ssh -i server2 disk.usage
```

Now let's install the cowsay package on all systems:
```bash
salt-ssh -i "*" pkg.install cowsay
```

And finally run a command on one of the machine:
```bash
salt-ssh -i server1 -r '/usr/games/cowsay "Hello me lad!"'
```

If you don't want to leave any trace on the system you can use the -W wipe option:
```bash
salt-ssh -i server1 -W -r '/usr/games/cowsay "Hello me lad!"'
```

## Agentless configuration management using salt-ssh

Since salt-ssh 2017 delivered with Ubuntu 18.04 is too old and has a bug with gitfs, the following
procedure has been done by installing a more recent version of salt-ssh using pip:
```bash
conda create -n salt
conda activate salt
conda install python=3.7
pip install salt-ssh
pip install pygit2
``` 

Here we'll install apache on all machines.

First clone the formula (you should have your own fork, see Salt Formulas above) since extra filerefs don't work on gitfs:
```bash
cd ~/salt_setup/salt/formulas
git clone https://github.com/saltstack-formulas/apache-formula.git
rm -rf ~/salt_setup/salt/formulas/apache-formula/.git
EOT
```

Declare the formula and required filerefs in your master configuration (~/salt_setup/master):
```yaml
file_roots:
  base:
    - /home/bruno/salt_setup/salt
    - /home/bruno/salt_setup/salt/formulas/apache-formula

extra_filerefs:
  - salt://apache/map.jinja
```

Download the example apache pillar:
```bash
curl -o ~/salt_setup/salt/pillar/apache.sls https://raw.githubusercontent.com/saltstack-formulas/apache-formula/master/pillar.example
```

Create the top.sls:
```bash
cat <<EOT >> ~/salt_setup/salt/top.sls
---
base:
  '*':
    - apache
EOT
```

Then go into your salt directory:
```bash
cd ~/salt_setup
```

Install apache on all machines:
```bash
salt-ssh '*' state.apply 
```

Test the installation of your apache:
```bash
curl server1/index.html
```

To uninstall apache, modify the top.sls:
Create the top.sls:
```yaml
---
base:
  '*':
    - apache.uninstall
```
Then apply the state on all machines:
```bash
salt-ssh '*' state.apply 
```

# Salt cloud

Salt can be used to manage (provisioning, starting, stopping) cloud instances (Amazon, Azure...)

(TBD)
