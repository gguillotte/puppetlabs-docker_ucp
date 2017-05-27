# docker_ucp

[![Puppet
Forge](http://img.shields.io/puppetforge/v/puppetlabs/docker_ucp.svg)](https://forge.puppetlabs.com/puppetlabs/docker_ucp)
[![Build
Status](https://travis-ci.org/puppetlabs/puppetlabs-docker_ucp.svg?branch=master)](https://travis-ci.org/puppetlabs/puppetlabs-docker_ucp)

1.  [Description - What the module does and why it is useful](#description)
2.  [Setup - The basics of getting started with docker_ucp](#setup)
    -  [Beginning with docker_ucp](#beginning-with-docker-ucp)
3.  [Usage - Configuration options and additional functionality](#usage)
4.  [Reference - An under-the-hood peek at what the module is doing and how](#reference)
5.  [Limitations - OS compatibility, etc.](#limitations)
6.  [Development - Guide for contributing to the module](#development)

## Description

The [Docker Universal Control
Plane](https://www.docker.com/products/docker-universal-control-plane) (UCP)
module installs and configures a UCP controller, and joins nodes to it.

This module provides a single class, `docker_ucp`, which uses the
official [`docker/ucp` container](https://hub.docker.com/r/docker/ucp/) to
bootstrap a UCP controller or join a node to an existing UCP-managed swarm.

## Setup

The module depends on Docker. To install and configure Docker with Puppet, use
the [`garethr/docker` module](https://forge.puppet.com/garethr/docker).

Review the
[UCP system requirements](https://docs.docker.com/datacenter/ucp/2.1/guides/admin/install/system-requirements/)
before installing a UCP controller on a node, and
[plan your UCP installation](https://docs.docker.com/datacenter/ucp/2.1/guides/admin/install/plan-installation/)
before deploying UCP in a production environment.

### Beginning with docker_ucp

The `docker_ucp` class has two modes of operation: to install and configure a
UCP controller on a node, and to join a node to a UCP-managed swarm.

#### Installing a controller

To install a UCP controller with the default `admin/orca` username and password,
add the `docker_ucp` class with the `controller` parameter set to true.

```puppet
class { 'docker_ucp':
  controller => true,
}
```

Once the controller is installed and available, log into it and change the admin user
password.

#### Joining a node

You can also use the `docker_ucp` class to join a node to a UCP-managed swarm.
The required class parameters depend on your UCP version. See
[Joining a node to a UCP-managed swarm](#joining-a-node-to-a-UCP-managed-swarm)
for examples.

## Usage

The parameters available to the docker_ucp class depend on your configuration. Consult the
[UCP CLI reference](https://docs.docker.com/datacenter/ucp/2.2/reference/cli/install/) for
details about these options.

```puppet
class { 'docker_ucp':
  controller                => true,
  host_address              => ::ipaddress_eth1,
  version                   => '1.0.0',
  usage                     => false,
  tracking                  => false,
  subject_alternative_names => ::ipaddress_eth1,
  external_ca               => false,
  swarm_scheduler           => 'binpack',
  swarm_port                => 19001,
  controller_port           => 19002,
  preserve_certs            => true,
  docker_socket_path        => '/var/run/docker.sock',
  license_file              => '/etc/docker/subscription.lic',
}
```

> **NOTE:** The `license_file` parameter requires UCP version 0.8.0 or newer.

### Joining a node to a UCP-managed swarm

#### Version =< 1

```puppet
class { 'docker_ucp':
  ucp_url     => 'https://ucp-controller.example.com',
  fingerprint => 'the-ucp-fingerprint-for-your-install',
}
```

The module uses the default username (`admin`) and password (`orca`). To set these,
provide those in parameters.

The class also takes other parameters useful for joining a node to a swarm. These should
also correspond with the options described in the official UCP documetation.

```puppet
class { 'docker_ucp':
  ucp_url                   => 'https://ucp-controller.example.com',
  fingerprint               => 'the-ucp-fingerprint-for-your-install',
  username                  => 'admin',
  password                  => 'orca',
  host_address              => ::ipaddress_eth1,
  subject_alternative_names => ::ipaddress_eth1,
  replica                   => true,
  version                   => '0.8.0',
  usage                     => false,
  tracking                  => false,
}
```

#### Version 2 and newer

In UCP version 2, Docker changed the underlying cluster scheduler from Swarm legacy to
Swarm mode, which also changed the join flags.

To join a node to a v2 manager (known as a controller in UCP v1):

```puppet
class { 'docker_ucp':
  version           => '2.1.0',
  token             => 'Your join token here',
  listen_address    => '192.168.1.2',
  advertise_address => '192.168.1.2',
  ucp_manager       => '192.168.1.1',
}
```

### Installing a Docker Trusted Registry

[Docker trusted registry](https://docs.docker.com/datacenter/dtr/2.2/guides/)
(DTR) is a containerized Docker image storage application that runs on a Docker UCP
cluster.

To install a DTR in a UCP cluster using the docker_ucp module, use the module's `dtr`
defined type.

```puppet
docker_ucp::dtr { 'Dtr install':
  install          => true,
  dtr_version      => 'latest',
  dtr_external_url => 'https://172.17.10.104',
  ucp_node         => 'ucp-04',
  ucp_username     => 'admin',
  ucp_password     => 'orca4307',
  ucp_insecure_tls => true,
  dtr_ucp_url      => 'https://172.17.10.101',
  require          => [ Class['docker_ucp']
}
```

In this example, we:

-   Set `install => true`, which configures a new registry.
-   Set `dtr_version`, which can be any version of the registry compatible with the UCP
    cluster.
-   Set `dtr_external_url` to the URL where the registry can be accessed.
-   Set `ucp_node` to the name of the node in the cluster where the registry will run.
-   Set the `ucp_username` and `ucp_password`.
-   Set `ucp_insecure_tls => true` to allow self-signed SSL certificates. In production,
    set this to false and use properly signed certificates.
-   Set `dtr_ucp_url` to the URL that the registry uses to contact the UCP cluster.

#### Joining a replica to a Docker Trusted Registry cluster

To
[join a replica](https://docs.docker.com/datacenter/dtr/2.2/guides/admin/install/#step-5-configure-dtr#step-7-join-replicas-to-the-cluster)
to a DTR cluster, use `join => true` instead of `install => true`.

```puppet
docker_ucp::dtr { 'Dtr install':
  join             => true,
  dtr_version      => 'latest',
  ucp_node         => 'ucp-03',
  ucp_username     => 'admin',
  ucp_password     => 'orca4307',
  ucp_insecure_tls => true,
  dtr_ucp_url      => 'https://172.17.10.101',
  require          => [ Class['docker_ucp']
}
```

Also, note that the replica doesn't use `dtr_external_url`.

> **NOTE:** You can't use the `install` and `join` parameters in the same declaration of
> a dtr defined type.

#### Removing a Docker Trusted Registry

To remove the DTR from your UCP cluster, provide the same parameters as in
[installation](#installing-a-docker-trusted-registry), but use `ensure => 'absent'`
instead of `install => true`.

```puppet
docker_ucp::dtr { 'Dtr install':
  ensure           => 'absent',
  dtr_version      => 'latest',
  dtr_external_url => 'https://172.17.10.104',
  ucp_node         => 'ucp-04',
  ucp_username     => 'admin',
  ucp_password     => 'orca4307',
  ucp_insecure_tls => true,
  dtr_ucp_url      => 'https://172.17.10.101',
}
```

## Reference

### Classes

-   `docker_ucp::docker_ucp`: Installs and manages the Docker Universal Control
    Plane (UCP) application using the official installer.

### Defined types

-   `docker_ucp::dtr`: Installs and manages a Docker Trusted Registry (DTR).

### `docker_ucp::docker_ucp`

#### Parameters

...

### `docker_ucp::docker_dtr`

#### Parameters

##### `install`

Data type: Boolean.

If true, attempt to install and configure a new DTR. Cannot be used in the same
declaration as the [`join`](#join) parameter.

Default: false.

##### `join`

Data type: Boolean.

If true, attempt to add a replica to a DTR cluster. Cannot be used in the same
declaration as the [`install`](#install) parameter.

Default: false.

##### `dtr_version`

Data type: String.

*Required*. The version of DTR to install.

Valid options: A string containing a valid DTR version number, such as '2.2.4', or
'latest'.

##### `dtr_external_url`

Data type: String.

*Required* if `dtr => true`. The external URL used to access this Docker Trusted Registry.

##### `ucp_node`

Data type: String.

*Required* if `dtr => true`. The UCP node on which the Docker trusted Registry will run.

##### `ucp_username`

Data type: String.

*Required* if `dtr => true`. The administrative user name for the UCP cluster.

##### `ucp_password`

Data type: String.

*Required* if `dtr => true`. The administrative password for the UCP cluster.

##### `ucp_insecure_tls`

Data type: Boolean.

Determines whether to check if the SSL certificate the UCP cluster is vaild. To use a
self-signed SSL certificate, set this to true.

Default: false.

##### `dtr_ucp_url`

Data type: String.

The URL that the DTR uses to communicate with the UCP cluster.

Default: undef.

##### `replica_id`

Data type: String.

The replica ID for the DTR cluster.

Default: undef.

##### `ucp_ca`

Data type: String.

The certificate authority to pass as part of the `install`/`join` flags.

Default: undef.

## Limitations

The docker_ucp module supports the same operating systems as UCP. As of Docker UCP 0.8,
this is limited to:

-   Red Hat Enterprise Linux 7.0, 7.1, and 7.2
-   Ubuntu 14.04 and 16.04
-   CentOS 7.1

## Development

Puppet modules on the Puppet Forge are open projects, and community contributions are
essential for keeping them great. Please follow our guidelines when contributing changes.

For more information, see our
[module contribution guide](https://docs.puppet.com/forge/contributing.html).

To see who's already involved, see the
[list of contributors](https://github.com/puppetlabs/puppetlabs-docker_platform/graphs/contributors).

### Maintainers

This module is maintained by:

-   Gareth Rushgrove <gareth@puppet.com>
-   Scott Coulton <scott.coulton@puppet.com>