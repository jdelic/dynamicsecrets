Salt Dynamic Secrets Module and Pillar
======================================

This module solves the *Initial Secret Introduction Problem* for Salt. You
assign secrets to nodes (or grains) and then use them in your states through the
``dynamicsecrets`` pillar. Secrets can be configured to represent

* cryptographically secure UUIDs,
* cryptographically secure strings of random characters suitable for passwords
* base64-encoded random bytes
* RSA keys
* or Consul ACL tokens

Secrets can be constant across the cluster or even differ for each salt-minion
("per-host").

The idea behind this is to solve the following problem: To manage secrets in a
cluster you have few options. You can pre-share them, which is a bad idea,
especially if those secrets are then unnecessarily shared across cluster
instances (multiple Salt installations using the same secrets). Also, your OPS
team might have access to all the secrets all the time, making them difficult to
rotate when someone leaves. On the other end of the spectrum you can use a tool
like `Hashicorp Vault <vault_>`__, but that raises the question of how you build
the cluster up to the point where a Vault instance is available.

Dynamicsecrets aims at exactly that small space inbetween installing your Salt
master and checking out your Salt configuration from your git repository and
installing your cluster to the point where Vault is available. A real-world
usage scenario for a Salt configuration that does exactly this is my very own
`Jdelic's Saltshaker <saltshaker_>`__.

The generated secrets are all kept in an *unencrypted* SQLite database in
``/etc/salt``. This is important, you **must** protect that database. Ideally,
you only use these secrets to bootstrap yourself into a cluster that then stores
and generates the more important secrets in a software like Vault.


Installation
------------
First, place the *Salt execution module* into your Salt configuration. Example
directory structure:

.. code-block::

    srv/salt-modules/modules/dynamicsecrets.py
    srv/salt-modules/pillar/dynamicsecrets.py
    srv/salt/top.sls
    srv/salt/...
    srv/pillar/top.sls
    srv/pillar/...

Then add a ``ext_pillar`` configuration to your salt-master:

.. code-block:: yaml

    # Extension modules
    extension_modules: /srv/salt-modules

    # set up access to Consul server
    dynamicsecrets.consul_url: http://169.254.1.1:8500/
    # You can either set a static token (not recommended)
    #     dynamicsecrets.consul_token: 12345678-abcd-...
    # or reference another dynamicsecrets secret as the ACL master token to use
    # to create new ACL tokens
    dynamicsecrets.consul_token_secret: consul-acl-master-token

    ext_pillar:
        - dynamicsecrets:
            config:
                approle-auth-token:
                    type: uuid
                concourse-encryption:
                    length: 32
                concourse-hostkey:
                    length: 2048
                    type: rsa
                consul-acl-token:
                    type: consul-acl-token
                    unique-per-host: True
                consul-acl-master-token:
                    type: uuid
                consul-encryptionkey:
                    encode: base64
                    length: 16
            grainmapping:
                roles:
                    authserver:
                        - approle-auth-token
            hostmapping:
                '*':
                    - consul-acl-token


In the above example, every node that has the grain ``roles:authserver`` can
access ``pillar['dynamicsecrets']['approle-auth-token']`` which is a UUID
constant over all salt-minions and every node can access
``pillar['dynamicsecrets']['consul-acl-token']`` which is a UUID that is
different for each salt-minion (and in my case used to create a Consul ACL for
each salt-minion by firing an event to the salt-master when the minion boots).

For ``type: password`` the Pillar will simply contain the random password
string.

For ``type: uuid`` the Pillar will return a UUID4 built from a secure random
source (as long as the OS provides one).

For ``type: rsa`` the Pillar will return a ``dict`` that has the following
properties:

* ``public_pem`` the public key in OpenSSL PEM encoding
* ``public`` the public key in `ssh-rsa` format
* ``key`` the private key in PEM encoding

For ``type: consul-acl-token`` the Pillar will return a ``dict`` that has the
following properties:

* ``accessor_id`` the accessor id of the ACL token (if ``firstrun`` is
  ``False``)
* ``secret_id`` the secret id of the ACL token (if ``firstrun`` is ``False``)
* ``firstrun`` a boolean flag that shows if the salt-master had a Consul server
  available to create ACL tokens. When a cluster is first started, this allows
  your Salt configuration to detect the chicken+egg problem of knowing when
  you're bootstrapping.


ConsuL ACL tokens
-----------------
If you want to use the Consul ACL token support in ``dynamicsecrets`` then your
salt-master **must** have access to a Consul server node and know a ACL master
token. ``dynamicsecrets`` talks directly to the Consul ACL API to create ACL
tokens with *no attached policy whatsoever*. You are then supposed to use Salt
to update the ACL tokens with your policies as they become available.

This is most easily done by using a Salt Reactor. An example can be found
`in this consul-acl Reactor <consul_reactor_>`__ and its associated
`salt-master configuration <reactor_config_>`__.


Usage
-----
As shown above, an `ext_pillar <ext_pillar_>`__ ends up in the ``pillar``
dictionary. Salt minions therefore get rendered pillars that can freely
reference ``pillar['dynamicsecrets']`` or ``__pillar__['dynamicsecrets']``,
depending on the use-case. On the salt-master, where the module is executed,
your code can also use the dynamicsecrets Salt execution module. So in
``pydsl`` states, reactors or in your own modules you can directly interface
with the module like this:

.. code-block:: python

    # get or create a secret for a specific host in a reactor
    # Note: in a reactor SLS, data['id'] is the salt-minion's ID
    salt['dynamicsecrets'].get_or_create(
        {
            "type": "uuid",
        },
        'consul-acl-token',
        host=data['id']
    )

    # get all secrets stored under a key (for all hosts)
    for sekrit in salt['dynamicsecrets'].loadall(
        'consul-acl-token):
        ...

    if salt['dynamicsecrets'].exists('consul-master-token',
        host="saltmaster"):
        ...


The Salt execution module can also be executed using the Salt client:

.. code-block:: shell

    $ salt 'saltmaster' dynamicsecrets.load consul-acl-token host=saltmaster


Future enhancements
-------------------
With a bit of work this could possibly use pysqlcipher to encrypt its backing
database.

.. _vault: https://vaultproject.io/
.. _saltshaker: https://github.com/jdelic/saltshaker/
.. _ext_pillar:
   https://docs.saltstack.com/en/latest/topics/development/external_pillars.html
.. _consul_reactor:
   https://github.com/jdelic/saltshaker/blob
   /231fc14c7521f44c83f76ad7de67fa062bd9aca8/srv/salt/orchestrate
   /consul-node-setup.sls
.. _reactor_config:
   https://github.com/jdelic/saltshaker/blob
   /231fc14c7521f44c83f76ad7de67fa062bd9aca8/etc/salt-master/master.d
   /saltshaker.conf#L131
