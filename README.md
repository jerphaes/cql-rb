# Ruby CQL3 driver

[![Build Status](https://travis-ci.org/iconara/cql-rb.png?branch=master)](https://travis-ci.org/iconara/cql-rb)
[![Coverage Status](https://coveralls.io/repos/iconara/cql-rb/badge.png)](https://coveralls.io/r/iconara/cql-rb)

# Requirements

Cassandra 1.2 or later with the native transport protocol turned on and a modern Ruby. It's tested continuously in Travis with Ruby 1.9.3, 2.0.0, and JRuby 1.7.x stable and head.

# Installation

    gem install cql-rb

## Configure Cassandra

If you're running Cassandra 1.2.5 or later the native transport protocol is enabled by default, if you're running an earlier version (but later than 1.2) you must enable it by editing `cassandra.yaml` and setting `start_native_transport` to `true`.

# Quick start

> Please note that all the examples in this document refer to the current release candidate of v1.1. If you want [the quick start for v1.0 you can find it here](https://github.com/iconara/cql-rb/tree/v1.0.6).

```ruby
require 'cql'

client = Cql::Client.connect(hosts: ['cassandra.example.com'])
client.use('system')
rows = client.execute('SELECT keyspace_name, columnfamily_name FROM schema_columnfamilies')
rows.each do |row|
  puts "The keyspace #{row['keyspace_name']} has a table called #{row['columnfamily_name']}"
end
```

The host you specify is just a seed node, the client will automatically connect to all other nodes in the cluster (or nodes in the same data center if you're running multiple rings).

When you're done you can call `#close` to disconnect from Cassandra:

```ruby
client.close
```

# Usage

The full [API documentation](http://rubydoc.info/gems/cql-rb/frames) is available from [rubydoc.info](http://rubydoc.info/).

## Changing keyspaces

```ruby
client.use('measurements')
```

or using CQL:

```ruby
client.execute('USE measurements')
```

## Running queries

You run CQL statements by passing them to `#execute`. Most statements don't have any result and the call will return nil.

```ruby
client.execute("INSERT INTO events (id, date, description) VALUES (23462, '2013-02-24T10:14:23+0000', 'Rang bell, ate food')")

client.execute("UPDATE events SET description = 'Oh, my' WHERE id = 13126")
```

If the CQL statement passed to `#execute` returns a result (e.g. it's a `SELECT` statement) the call returns an enumerable of rows:

```ruby
rows = client.execute('SELECT date, description FROM events')
rows.each do |row|
  row.each do |key, value|
    puts "#{key} = #{value}"
  end
end
```

The enumerable also has an accessor called `metadata` which returns a description of the rows and columns:

```ruby
rows = client.execute('SELECT date, description FROM events'
rows.metadata['date'].type # => :date
```

Each call to `#execute` selects a random connection to run the query on.

## Creating keyspaces and tables

There is no special facility for creating keyspaces and tables, they are created by executing CQL:

```ruby
keyspace_definition = <<-KSDEF
  CREATE KEYSPACE measurements
  WITH replication = {
    'class': 'SimpleStrategy',
    'replication_factor': 3
  }
KSDEF

table_definition = <<-TABLEDEF
  CREATE TABLE events (
    id INT,
    date DATE,
    comment VARCHAR,
    PRIMARY KEY (id)
  )
TABLEDEF

client.execute(keyspace_definition)
client.use('measurements')
client.execute(table_definition)
```

You can also `ALTER` keyspaces and tables, and you can read more about that in the [CQL3 syntax documentation](https://github.com/apache/cassandra/blob/cassandra-1.2/doc/cql3/CQL.textile).

## Prepared statements

The driver supports prepared statements. Use `#prepare` to create a statement object, and then call `#execute` on that object to run a statement. You must supply values for all bound parameters when you call `#execute`.

```ruby
statement = client.prepare('SELECT date, description FROM events WHERE id = ?')
rows = statement.execute(1235)
```

A prepared statement can be run many times, but the CQL parsing will only be done once. Use prepared statements for queries you run over and over again.

`INSERT`, `UPDATE`, `DELETE` and `SELECT` statements can be prepared, other statements may raise `QueryError`.

Statements are prepared on all connections and each call to `#execute` selects a random connection to run the query on.

## Consistency

You can specify the default consistency to use when you create a new `Client`:

```ruby
client = Cql::Client.connect(hosts: %w[localhost], consistency: :all)
```

The `#execute` (of `Client` and `PreparedStatement`) method also supports setting the desired consistency level on a per-request basis:

```ruby
client.execute('SELECT * FROM peers', consistency: :local_quorum)
```

for backwards compatibility with v1.0 you can also pass the consistency as just a symbol:

```ruby
client.execute('SELECT * FROM peers', :local_quorum)
```

The possible values for consistency are: 

* `:any`
* `:one`
* `:two`
* `:three`
* `:quorum`
* `:all`
* `:local_quorum`
* `:each_quorum`

The default consistency level unless you've set it yourself is `:quorum`.

Consistency is ignored for `USE`, `TRUNCATE`, `CREATE` and `ALTER` statements, and some (like `:any`) aren't allowed in all situations.

# CQL3

This is just a driver for the Cassandra native CQL protocol, it doesn't really know anything about CQL. You can run any CQL3 statement and the driver will return whatever Cassandra replies with.

Read more about CQL3 in the [CQL3 syntax documentation](https://github.com/apache/cassandra/blob/cassandra-1.2/doc/cql3/CQL.textile) and the [Cassandra query documentation](http://www.datastax.com/docs/1.2/cql_cli/querying_cql).

# Cassandra 2.0

Cassandra 2.0 introduced a new version of the native protocol with some new features like argument interpolation in non-prepared statements, result set cursors, a new authentication mechanism and the `SERIAL` consistency. These features are not yet supported, but the driver will work with Cassandra 2.0 using the earlier protocol.

# Troubleshooting

## I get "Deadlock detected" errors

This means that the driver's IO reactor has crashed hard. Most of the time it means that you're using a framework, server or runtime that forks and you call `Client.connect` in the parent process. Check the documentation and see if there's any way you can register to run some piece of code in the child process just after a fork, and connect there.

This is how you do it in Resque:

```ruby
Resque.after_fork = proc do
  # connect to Cassandra here
end
```

and this is how you do it in Passenger:

```ruby
PhusionPassenger.on_event(:starting_worker_process) do |forked|
  if forked
    # connect to Cassandra here
  end
end
```

in Unicorn you do it in the config file:

```ruby
after_fork do |server, worker|
  # connect to Cassandra here
end
```

If your process does not fork and you still encounter deadlock errors, it might also be a bug. All IO is done is a dedicated thread, and if something happens that makes that thread shut down, Ruby will detect that the locks that the client code is waiting on can't be unlocked.

## I'm not getting all elements back from my list/set/map

There's a known issue with collections that get too big. The protocol uses a short for the size of collections, but there is no way for Cassandra to stop you from creating a collection bigger than 65536 elements, so when you do the size field overflows with strange results. The data is there, you just can't get it back.

## Authentication doesn't work

Please open an issue. It should be working, but it's hard to write tests for, so there may be edge cases that aren't covered.

## I'm connecting to port 9160 and it doesn't work

Port 9160 is the old Thrift interface, the binary protocol runs on 9042. This is also the default port for cql-rb, so unless you've changed the port in `cassandra.yaml`, don't override the port.

## Something else is not working

Open an issue and someone will try to help you out. Please include the gem version, Casandra version and Ruby version, and explain as much about what you're doing as you can, preferably the smallest piece of code that reliably triggers the problem. The more information you give, the better the chances you will get help.

# Changelog & versioning

Check out the [releases on GitHub](https://github.com/iconara/cql-rb/releases). Version numbering follows the [semantic versioning](http://semver.org/) scheme.

Private and experimental APIs, defined as whatever is not in the [public API documentation](http://rubydoc.info/gems/cql-rb/frames) will change without warning. If you've been recommended to try an experimental API by the maintainers, please let them know if you depend on that API. Experimental APIs will eventually become public, and knowing how they are used helps in determining their maturity.

Prereleases will be stable, in the sense that they will have finished and properly tested features only, but may introduce APIs that will change before the final release. Please use the prereleases and report bugs, but don't deploy them to production without consulting the maintainers, or doing extensive testing yourself. If you do deploy to production please let the maintainers know as this helps determining the maturity of the release.

# Known bugs & limitations

* No support for compression.
* No support for request tracing.
* JRuby 1.6 is not officially supported, although 1.6.8 should work, if you're stuck in JRuby 1.6.8 try and see if it works for you.
* Large results are buffered in memory until the whole response has been loaded, the protocol makes it possible to start to deliver rows to the client code as soon as the metadata is loaded, but this is not supported yet.
* There is no cluster introspection utilities (like the `DESCRIBE` commands in `cqlsh`).
* New features in v2 of the protocol are not supported

Also check out the [issues](https://github.com/iconara/cql-rb/issues) for open bugs.

# How to contribute

Fork the repository, make your changes in a topic branch that branches off from the right place in the history (HEAD isn't necessarily always right), make your changes and finally submit a pull request.

Follow the style of the existing code, make sure that existing tests pass, and that everything new has good test coverage. Put some effort into writing clear and concise commit messages, and write a good pull request description.

It takes time to understand other people's code, and even more time to understand a patch, so do as much as you can to make the maintainers' work easier. Be prepared for rejection, many times a feature is already planned, or the proposed design would be in the way of other planned features, or the maintainers' just feel that it will be faster to implement the features themselves than to try to integrate your patch.

Feel free to open a pull request before the feature is finished, that way you can have a conversation with the maintainers' during the development, and you can make adjustments to the design as you go along instead of having your whole feature rejected because of reasons such as those above. If you do, please make it clear that the pull request is a work in progress, or a request for comment.

Always remember that the maintainers' work on this project in their free time and that they don't work for you, or for your benefit. They have no obligation to do what you think is right -- but if you're nice they might anyway.

# Copyright

Copyright 2013 Theo Hultberg/Iconara and contributors

_Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License You may obtain a copy of the License at_

[http://www.apache.org/licenses/LICENSE-2.0](http://www.apache.org/licenses/LICENSE-2.0)

_Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License._