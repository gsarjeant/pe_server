# pe_server

## Overview

pe_server is designed to aide in the setup and management of a split-stack
Puppet Enterprise infrastructure.  This includes full-stack masters, such as in
a standby/DR type of configuration, or split installs of various configurations.

This does not modify or replace the Puppet Enterprise installer or do anything
that isn't supported.

Basically, this module ensures the various services and components of a
Puppet Enterprise stack point to the right places, have the right certificates,
the right access, and the appropriate configurations accordingly.  For proper
use, this relies on PE's answer files to provide the correct information at
installation time, as well as a bootstrapping procedure for standing the servers
up.

This is still being tweaked and documented.  See the "Contributions" section
below.

## Example Usage

I have a sample "control repository" available at
[https://github.com/joshbeard/puppet-tri-stack](https://github.com/joshbeard/puppet-tri-stack)
that uses this module to stand up a 3+x3+ PE stack.

This includes one full
"primary" stack that has a CA/Master, a PuppetDB/PostgreSQL on a separate host,
and a Console/PostgreSQL on a separate host.  A secondary stack that includes
a Standby CA/Master, a PuppetDB/standby PostgreSQL on a separate host, and a
Console/standby PostgreSQL on a separate host, with additional compile masters
as an option.


## Classes

###pe_server

The base class. Configures the settings for an agent's CA server, filebucket,
and has the ability to export resources for addition to a PuppetDB and Console
whitelist.

####Parameters:

#####`is_master`

Whether the node is a Puppet master or not.  This simply changes the filebucket
server in the global `site.pp` to point to the specified filebucket server.
Valid values are `true` or `false`.  Defaults to `false`

#####`ca_server`

If specified, this will configure the `ca_server` option in `puppet.conf` to point
to the value of this parameter. Value should be a resolvable address to the
CA server.  Defaults to `undef`

#####`filebucket_server`

Should be set to a resolvable address to a filebucket server. Defaults to
`$::settings::server`

#####`change_filebucket`

Specifies whether to configure the `archive_file_server` setting in `puppet.conf`
Additionally, if `is_master` is set to `true`, this will set the filebucket
server in the global `site.pp` (`$confdir/manifests/site.pp`) Defaults to
`true`

#####`export_puppetdb_whitelist`

Specifies whether to export the `$::clientcert` as an entry for the PuppetDB
whitelist, which can optionally be collected by a PuppetDB server (via the
pe_server::puppetdb class)
Valid values
are `true` or `false`.  Defaults to `true`

#####`export_console_authorization`

Specifies whether to export the `$::clientcert` as an entry for the Console
authorization, which can optionally be collected by a Console server (via the
pe_server::console class)
Valid values
are `true` or `false`.  Defaults to `true`

###pe_server::ca

Class used for configuring a Puppet Enterprise CA server.

####Parameters

#####`active_ca`

Whether this is an active CA or not.  If it is set to `false`, the `ca`
option in the `puppet.conf` file will be set to false, and the HTTP SSL revocation
certificate will be modified.

#####`autosign`

An optional array of certificate names to add to the CA's `autosign.conf` file.

#####`generate_certs`

Takes a hash of certificates to generate, with optional `dns_alt_names`. This
uses [Reid Vandewiele's](https://github.com/reidmv)
[puppet_certificate](https://forge.puppetlabs.com/reidmv/puppet_certificate)
module and simply passes the hash onto that.  See his module for information
on what the hash can look like.

This will generate the specified certificates on this CA server.  This can be
useful for cases where the SSL data will be synchronized and replaced on a
secondary CA server.

As an example:

```puppet
generate_certs => {
  'puppetca02.example.com' => {
    dns_alt_names => [
      'puppetca02',
      'puppetca.example.com',
      'puppetca',
    ],
  },
}
```


###pe_server::console

Used for configuring the Puppet Enterprise Console and managing its certificates.

####Parameters

#####`ca_server`

Required. Specifies a resolvable address for the CA server for configuring  the
console's settings.

#####`console_cert_name`

The certificate name for use with the PE console.  This is useful for having
multiple consoles with different certificate names.  This will configure a
console's httpd configuration to use this certificate name, the event inspector,
and the console's settings. If `console_certs_from_ca` is set to `true`,
certificates with this name will attempted to be copied from the CA.  If
`create_console_certs` is set to `true`, certificates will be created on the
console server with this name.
Defaults to `pe-internal-dashboard`.

#####`cert_owner`

The owner of the certificate files on the console.  Defaults to
`puppet-dashboard`

#####`cert_group`

The group of the certificate files on the console.  Defaults to
`puppet-dashboard`

#####`inventory_server`

Specifies a value for `inventory_server` in the console's settings.
Defaults to `$::settings::server`

#####`puppetdb_host`

Specifies a resolvable address to a PuppetDB instance. Defaults to
`$::fqdn`

#####`puppetdb_port`

Specifies the port that a PuppetDB instance is listening on. Defaults to
`8081`

#####`create_console_certs`

Specifies whether console certs should be created locally using the dashboard's
rake tasks.  This will create the certificates, send a CSR to the CA, and
retrieve the signed certificates.  Defaults to `true` (this is normal
behavior).

#####`console_certs_from_ca`

Specifies whether the console certs should be copied from the CA.  This will
look for certificate names matching the `console_cert_name` in on the CA in
`/opt/puppet/share/puppet-dashboard/certs/`.  This setting and
`create_console_certs` are mutually exclusive.  Valid values are `true` and
`false`.  The default is `false`

#####`collect_exported_authorization`

Specifies whether exported resources for the console authorization should be
collected and realized.  Valid values are `true` and `false`.  The default
is `true`.

###pe_server::puppetdb

For managing PuppetDB instances and PE's PostgreSQL instance.

####Parameters

#####`manage_postgres`

Specifies whether PostgreSQL should be managed by this class via the
`pe_puppetdb::pe` class.  If true, the `postgres_listen_addresses` and
`database_host` parameters will be set on the `pe_puppetdb::pe` class.
Valid values are `true` and `false`.  The default is `true`

#####`postgres_listen_address`

The listen address for the PostgreSQL instance.  This is passed as the
`postgres_listen_addresses` parameter to the `pe_puppetdb::pe` class.
If the PostgreSQL database will need to be reached from another host, you'll
need to use this parameter to ensure it's listening on the desired interfaces.
The default value is `*` for all interfaces.

#####`postgres_database_host`

Specifies the host that the PuppetDB PostgreSQL database can be reached at.
This defaults to `$::fqdn`

#####`puppetdb_ssl_setup`

Specifies whether the `puppetdb ssl-setup` should be ran to create PuppetDB
SSL certificates (This is `/opt/puppet/sbin/puppetdb ssl-setup`). This will
compare the certificates in `$ssldir` against the PuppetDB certificates and
run the ssl-setup if they don't match.  The PuppetDB ssl-setup needs to be ran
when the node's certificates change.  During installation of PE, this is done
for you.  However, if you recreate certificates after installation or copy the
  certificates from another host, you'll need to run the PuppetDB ssl-setup.
Valid values are `true` and `false`.
The default is `true`

#####`collect_exported_whitelist`

Specifies whether exported resources for the PuppetDB whitelist should be
collected and realized.  Valid values are `true` and `false`.  The default
is `true`.

###pe_server::mcollective

#### Parameters

#####`primary`

Address for the "primary/source of truth" server that has the Mcollective
certificates to sync.  Defaults to `::$settings::server`

#####`shared_credentials`

Whether to synchronize the "credentials" file from the master.  This is the
`/etc/puppetlabs/mcollective/credentials` file.  Defaults to `true`

#####`activemq_brokers`

Comma-separated list of ActiveMQ brokers.  This is passed along to the built-in
`pe_mcollective::role::master` class, which is declared by this class.

#####`stomp_servers`

Comma-separated list of Mcollective STOMP servers.  This is passed along to the
built-in `pe_mcollective::role::master` class, which is declared by this class.

### pe_server::console::database

This configures the `/etc/puppetlabs/puppet-dashboard/database.yml` file,
providing connection details for the console's PostgreSQL database.

It also configures the `/etc/puppetlabs/console-auth/database.yml` file,
providing connection details for the console_auth PostgreSQL database.

#### Parameters

#####`password`

The PostgreSQL database password for the console.  This parameter is required.

#####`console_auth_password`

The PostgreSQL database password for the console_auth database.  This parameter
is required.

#####`database`

The name of the console database. Defaults to `console`

#####`username`

The PostgreSQL database username for the console.  Defaults to `console`

#####`host`

The resolvable address to the PostgreSQL instance with the console database.
Defaults to `localhost`

#####`port`

The port that the console's PostgreSQL instance can be reached at.  Defaults to
`5432`

#####`adapter`

The database adapter to use for connecting to the database.  Defaults to
`postgresql`

#####`console_auth_database`

The name of the console_auth PostgreSQL database.  Defaults to `console_auth`

#####`console_auth_username`

The username for the console_auth PostgreSQL database.  Defaults to `console_auth`

###pe_server::console::event_inspector

This is a private class (it's called by the console class - not you.)
This class configures the console's event inspector config file at
`/opt/puppet/share/event-inspector/config/config.yml`

####Parameters

#####`console_cert_name`

The certificate name of the console.  Defaults to `$::fqdn`

#####`puppetdb_port`

The port for reaching the specified PuppetDB instance.  Defaults to `8081`

#####`puppetdb_host`

The host for reaching the PuppetDB instance.  Defaults to `$::fqdn`

## Defined Types

###pe_server::puppetdb::whitelist

This is used to add certificate names to the PuppetDB whitelist at
`/etc/puppetlabs/puppetdb/certificate-whitelist`

#### Parameters

#####`certname`

The certname to add to the PuppetDB whitelist. Defaults to `$name`

#####`match`

An optional string or regular expression used to match the entry in the
PuppetDB whitelist file.  Defaults to `$name`

###pe_server::console::authorization

This is used to add certificate names to the PE Console authorization at
`/etc/puppetlabs/console-auth/certificate_authorization.yml`

#### Parameters

#####`certname`

The certname to add to the Console whitelist. Defaults to `$name`

#####`match`

An optional string or regular expression used to match the entry in the
Console whitelist file.  Defaults to `$name`

#####`role`

The authorization role for the certificate. Valid values are `read-write`,
`read-only`, or `admin`.  Defaults to `read-write`

## Compatibility

This module is specific to Puppet Enterprise.  Puppet Open Source environments
are not supported.

This has been tested and developed against Puppet Enterprise 3.0 - 3.3

## Contributions

Contributions are *very* welcomed.  Here's a few things, off the top of my head,
that I think needs a look:

* Test in various combinations of install. E.g. two full stack masters.
* Can we safely manage more things?  Mcollective and ActiveMQ isn't really well
managed by this module.  Can it be improved without stomping on PE's toes?
* Compatibility with future versions of PE (3.4).  Certain parts of this may
be unneeded with upgrades, some things will need changing, some things will need
to be added.
* Cleaner?  There's some parts of this that contains dragons.  Where can it be
improved?  And the top design goal here is to *not* conflict with the out-of-the
box PE installer and its modules.  This should just add a *safe* layer on top
of it.
* Documentation (as usual)

## Authors

[Josh Beard](http://signalboxes.net)

This module is heavily derived from [Tom Linkin's](https://github.com/trlinkin)
[pe_secondary](https://github.com/trlinkin/pe_secondary) module.
