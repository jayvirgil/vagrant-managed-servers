# Vagrant ManagedServers Provider

[![Build Status](https://travis-ci.org/tknerr/vagrant-managed-servers.png?branch=master)](https://travis-ci.org/tknerr/vagrant-managed-servers)

This is a [Vagrant](http://www.vagrantup.com) 1.2+ plugin that adds a provider for "managed servers" to Vagrant, i.e. servers for which you have SSH access but no control over their lifecycle.

Since you don't control the lifecycle:
 * `up` and `destroy` are re-interpreted as "linking" / "unlinking" vagrant with a managed server
 * once "linked", the `ssh` and `provision` commands work as expected and `status` shows the managed server as either "running" or "not reachable"
 * `halt`, `reload` and `suspend` and `resume` are no-ops in this provider

Credits: this provider was initially based on the [vagrant-aws](https://github.com/mitchellh/vagrant-aws) provider with the AWS-specific functionality stripped out.

**NOTE:** This plugin requires Vagrant 1.2+

## Features

* SSH into managed servers.
* Provision managed servers with any built-in Vagrant provisioner.
* Minimal synced folder support via `rsync`.

## Usage

Install using the standard Vagrant plugin installation method:
```
$ vagrant plugin install vagrant-managed-servers
```

In the Vagrantfile you can now use the `managed` provider and specify the managed server's hostname and credentials:
```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "tknerr/managed-server-dummy"

  config.vm.provider :managed do |managed, override|
    managed.server = "foo.acme.com"
    override.ssh.username = "bob"
    override.ssh.private_key_path = "/path/to/bobs_private_key"
  end
end
```

Next run `vagrant up --provider=managed` in order to "link" the vagrant VM with the managed server: 
```
$ vagrant up --provider=managed
Bringing machine 'default' up with 'managed' provider...
==> default: Box 'tknerr/managed-server-dummy' could not be found. Attempting to find and install...
    default: Box Provider: managed
    default: Box Version: >= 0
==> default: Loading metadata for box 'tknerr/managed-server-dummy'
    default: URL: https://vagrantcloud.com/tknerr/managed-server-dummy
==> default: Adding box 'tknerr/managed-server-dummy' (v1.0.0) for provider: managed
    default: Downloading: https://vagrantcloud.com/tknerr/managed-server-dummy/version/1/provider/managed.box
    default: Progress: 100% (Rate: 122k/s, Estimated time remaining: --:--:--)
==> default: Successfully added box 'tknerr/managed-server-dummy' (v1.0.0) for 'managed'!
==> default: Linking vagrant with managed server foo.acme.com
==> default:  -- Server: foo.acme.com
```

Once linked, you can run `vagrant ssh` to ssh into the managed server or `vagrant provision` to provision that server with any of the available vagrant provisioners:
```
$ vagrant provision
...
$ vagrant ssh
...
``` 

If you are done, you can "unlink" vagrant from the managed server by running `vagrant destroy`:
```
$ vagrant destroy -f
==> default: Unlinking vagrant from managed server foo.acme.com
==> default:  -- Server: foo.acme.com
```

If you try any of the other VM lifecycle commands like `halt`, `resume`, `reload`, etc... you will get a warning that these commands are not supported with the vagrant-managed-servers provider. 

## Box Format

Every provider in Vagrant must introduce a custom box format. This provider introduces a "dummy box" for the `managed` provider which is really nothing more than the required `metadata.json` with the provider name set to "managed". 

For Vagrant 1.5+ you can use the [tknerr/managed-server-dummy](https://vagrantcloud.com/tknerr/managed-server-dummy) vagrantcloud box:
```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "tknerr/managed-server-dummy"
  ...
end
```

For Vagrant < 1.5 you can point to the [dummy.box](https://github.com/tknerr/vagrant-managed-servers/raw/master/dummy.box) URL directly:
```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "managed-server-dummy"
  config.vm.box_url = "https://github.com/tknerr/vagrant-managed-servers/raw/master/dummy.box"
  ...
end
```

## Configuration

This provider currently exposes only a single provider-specific configuration option:

* `server` - The IP address or hostname of the existing managed server

It can be set like typical provider-specific configuration:

```ruby
Vagrant.configure("2") do |config|
  # ... other stuff

  config.vm.provider :managed do |managed|
    managed.server = "some-server.org"
  end
end
```

## Networks

Networking features in the form of `config.vm.network` are not
supported with `vagrant-managed-servers`. If any of these are
specified, Vagrant will emit a warning and just ignore it.

## Synced Folders

There is minimal support for synced folders. Upon `vagrant provision`, 
the managed servers provider will use
`rsync` (if available) to uni-directionally sync the folder to
the remote machine over SSH.

This is good enough for all built-in Vagrant provisioners (shell,
chef, and puppet) to work!

## Development

To work on the `vagrant-managed-servers` plugin, clone this repository out, and use
[Bundler](http://gembundler.com) to get the dependencies:

```
$ bundle
```

Once you have the dependencies, verify the unit tests pass with `rake`:

```
$ bundle exec rake
```

If those pass, you're ready to start developing the plugin. You can test
the plugin without installing it into your Vagrant environment by using the
`Vagrantfile` in the top level of this directory and use bundler to execute Vagrant.

First, fake a managed server by bringing up the `fake_managed_server` vagrant VM with the default virtualbox provider:

```
$ bundle exec vagrant up fake_managed_server
```

Now you can use the managed provider (defined in a separate VM named `my_server`) to ssh into or provision the (fake) managed server:

```
$ # link vagrant with the server
$ bundle exec vagrant up my_server --provider=managed
$ # ssh / provision
$ bundle exec vagrant ssh my_server
$ bundle exec vagrant provision my_server
$ # unlink
$ bundle exec vagrant destroy my_server
```
