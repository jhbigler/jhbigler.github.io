---
layout: post
title:  "Building the Vector Puppet Module"
---

[Vector](https://vector.dev) is a system observability and telemetry tool written in Rust that specializes in collecting, processing, and shipping logs and metrics in a vendor-neutral fashion. You can think of it as akin to tools like Logstash and Elastic's Beats software (Filebeat, Metricbeat, etc), Fluent Bit/Fluentd, and many others. Those tools are fine for their purpose. However I find many of them either limited, or simply difficult to learn and set up, often containing strange configuration syntax. As such, I stumbled onto Vector and quickly appreciated it's simple syntax, small resource footprint, and ability to transform logs and metrics using its VRL (Vector Remap Language) processor.

At that time, there was no open-source Puppet module to install and configure Vector. I figured I would address that gap, and in this post I will demonstrate the thought process in creating the [Vector module](https://github.com/jhbigler/puppet-vector). I won't explain every single bit of code or go over every single variable. The code expressed doesn't exactly represent what is currently in the repository either, it is a simplified version.

In general, when managing an application for a system, there often three primary stages:

1. Installation
2. Configuration
3. Execution

Some applications require additional steps, but this is a good starting point. Puppet classes can easily be broken down into ordered steps using sub classes and the `contain` directive:

```puppet
# manifests/init.pp
class vector {
    contain vector::install
    contain vector::configure
    contain vector::service

    Class['vector::install']
    -> Class['vector::configure']
    -> Class['vector::service']
}
```

Now that we have the initial framework in place, we need to write the three sub classes.

## Installation

The first step is install vector. While we *could* attempt to configure package repositories on the system to ensure Vector is available, this would be very tricky when considering all of types of packagement management systems that could be used (apt, snap, yum/dnf, etc.), and it's also possible that the user is wanting to install from mirrors and/or custom repositories. Instead of polluting the module with a large list of configuration items and spaghetti code to configure all of that, I opted to leave package repository management out of scope of the module. There are plenty of puppet modules that can configure those things much better than I can.

With that decided, we instead have the `vector::install` simply use the `package` resource to install it and assume the user has already configure relevant repositories:

```puppet
# manifests/install.pp
class vector::install(
    String $vector_ensure = present,
) {
    package { 'vector':
        ensure => $vector_ensure,
    }
}
```

By default, this simply ensures Vector is installed. The user can override `vector::install::vector_ensure` to a specific version to make sure that specific version is installed.

## Configuration

Although Vector is relatively easy to configure, the `vector::configure` class will still be the most involved. Vector has three main components in its operation:

- __Sources__: Where data is generated (log file, kubernetes logs, journald, socket, etc)
- __Transforms__: Where data is manipulated (usually using vrl)
- __Sinks__: Where data is output (Elasticsearch, Kafka, S3, etc)

The user plugs these pieces together to create a topology. In addition, there are global options, such as api settings and the location of vector's data directory.

Most people configure all of these in a single configuration file (such as /etc/vector/vector.toml). However, I want sources, transforms, and sinks to be declared as puppet resources, so that users may declare them across multiple modules. Coordinating all of that into a single file is tricky - the `concat` module could come into play, but I try to lessen dependencies on other modules as much as possible, and that module is not guaranteed to work well with JSON or YAML data (Vector config files can be TOML, JSON, or YAML).

Instead, we will take advantage of a little-known (and not very well advertised) feature in vector configuration: [Automatic Namespacing](https://vector.dev/docs/reference/configuration/#automatic-namespacing). This feature allows us to split up configurations by their types into different files. That way, when the user declares a source, transform, or sink, a new file is simply put into its corresponding directory.

The approach will be as such:

- Global configurations will go into `/etc/vector/global.yaml`
- Puppet will create `/etc/vector/configs` and `/etc/vector/configs/{sources, transforms, sinks}`
- We create defined types for each of the vector components, which will craft files to go into their corresponding directory

```puppet
# manifests/configure.pp
class vector::configure(
    Hash $global_opts = {},
) {
    $global_opts_file = '/etc/vector/global.yml'
    $configs_dir = '/etc/vector/configs'

    $sources_dir = "${configs_dir}/sources"
    $transforms_dir = "${configs_dir}/transforms"
    $sinks_dir = "${configs_dir}/sinks"

    # Craft the global options using stdlib's to_yaml function
    file { $global_opts_file:
        ensure  => file,
        content => to_yaml($global_opts),
        notify  => Service['vector'],
    }

    file { $configs_dir:
        ensure  => directory,
        recurse => true,
        purge   => true,
    }
    # Yes, you can define multiple directories at the same time!
    -> file { [$sources_dir, $transforms_dir, $sinks_dir]:
        ensure  => directory,
        recurse => true,
        purge   => true,
    }

    # Systemd service file
    file { '/etc/systemd/system/vector.service':
        ensure  => file,
        content => epp('vector/vector.service.epp'),
        notify  => Class['vector::service'],
    }

    # Set up automatic dependencies of the configurations dirs and the resource types
    File[$sources_dir, $transforms_dir, $sinks_dir] -> Vector::Source<||> ~> Class['vector::service']
    File[$sources_dir, $transforms_dir, $sinks_dir] -> Vector::Sink<||> ~> Class['vector::service']
    File[$sources_dir, $transforms_dir, $sinks_dir] -> Vector::Transform<||> ~> Class['vector::service']
}
```

Now that we have the directory structure in place, we need the API for users to programmably create sources, transforms, and sinks as needed. There are many types of sources, sinks, and transforms, each with their own configuration options. Maintaining a full list of them in Puppet is unrealistic since new types get added, configuration options are changed, and so on. Instead, we simply provide generic resource types for them. Let's look at the requirements for each type:

- A __source__ must have the `type` field, as well other fields depending on the source type
- A __transform__ must have the `type` field, an `inputs` array, and other fields depending on the type
- A __sink__ is much like a transform - it must have the `type` field, an `inputs` array, and other fields depending on the type

Each of them must also be named, which we can simply use the resource `$title` variable to define that - doing so ensures there are no duplicate names in the same resource type. Also, since we simply need to create a file for each of these resources, there's no need to crack open Puppet's Ruby API documentation - using Puppet's defined resource will do the trick. Let's see what they look like:

```puppet
# manifests/source.pp
define vector::source (
  String                    $type,
  Hash                      $parameters,
  Vector::ValidConfigFormat $format = 'toml',
) {

  $source_hash = $parameters + { 'type' => $type }

  $source_file_name = "${vector::configure::sources_dir}/${title}.${format}"

  file { $source_file_name:
    ensure  => file,
    content => vector::dump_config($source_hash, $format),
  }
}

# manifests/transform.pp
define vector::transform (
  String                    $type,
  Array[String]             $inputs,
  Hash                      $parameters,
  Vector::ValidConfigFormat $format = 'toml',
) {

  $transform_hash = $parameters + { 'type' => $type, 'inputs' => $inputs }

  $transform_file_name = "${vector::configure::transforms_dir}/${title}.${format}"

  file { $transform_file_name:
    ensure  => file,
    content => vector::dump_config($transform_hash, $format),
  }
}

# manifests/sink.pp
define vector::sink (
  String                    $type,
  Array[String]             $inputs,
  Hash                      $parameters,
  Vector::ValidConfigFormat $format    = 'toml',
) {

  $sink_hash = $parameters + { 'type' => $type, 'inputs' => $inputs }

  $sink_file_name = "${vector::configure::sinks_dir}/${title}.${format}"

  file { $sink_file_name:
    ensure  => file,
    content => vector::dump_config($sink_hash, $format),
  }
}
```

The idea with all three types is the same: Create a configuration hash, craft the full path using the name of the resource and the provided format, and dump the hash according the provided format. All three use a helper function called `vector::dump_config`:

```puppet
# functions/dump_config.pp
function vector::dump_config(
  Hash $data,
  Vector::ValidConfigFormat $format = 'toml',
) >> String {
  case $format {
    # Accept 'yaml' and 'yml' for yaml data
    /ya?ml/ : { to_yaml($data) }
    'toml'  : { to_toml($data) }
    # Assume everything else is JSON
    default : { to_json($data) }
  }
}
```

They all also reference a custom data type `Vector::ValidConfigFormat`, to ensure the file extension is one that Vector recognizes:

```puppet
# types/validconfigformat.pp
type Vector::ValidConfigFormat = Enum['json','yaml','yml','toml']
```

## Execution/Service

Finally, we create the `vector::service` class. This class ends up being very simple - just make sure it is running:

```puppet
# manifests/service.pp
class vector::service {
    service { 'vector':
        ensure => running,
        enable => true,
    }
}
```

And that's pretty much it! I skipped over many steps, such as what the systemd service file actually looks like. There are any more options in the actual module, plus some variables, steps, etc have been added or moved since I wrote the first version of the module. Definitely consult the repository to see what the code currently looks like!

## Simple example usage

```puppet
node default {
    include vector

    vector::source { 'logfile_input':
        type       => 'file',
        parameters => {
            'include' => ['/var/log/**/*.log'],
        },
    }

    vector::transform { 'logfile_transform':
        type       => 'remap',
        inputs     => ['*'],
        parameters => {
            'source' => '.foo = "bar"',
        },
    }

    vector::sink { 'logfile_kafka':
        type       => 'kafka',
        inputs     => ['logfile_transform'],
        parameters => {
            'bootstrap_servers' => 'localhost:9092',
            'topic'             => 'logs',
            'encoding'          => {
                'codec' => 'json',
            },
        }
    }
}
```