# KSUID for Ruby

[![Build Status](https://github.com/michaelherold/ksuid-ruby/workflows/Continuous%20integration/badge.svg)][actions]
[![Test Coverage](https://api.codeclimate.com/v1/badges/94b2a2d4082bff21c10f/test_coverage)][test-coverage]
[![Maintainability](https://api.codeclimate.com/v1/badges/94b2a2d4082bff21c10f/maintainability)][maintainability]
[![Inline docs](http://inch-ci.org/github/michaelherold/ksuid-ruby.svg?branch=master)][inch]

[actions]: https://github.com/michaelherold/ksuid-ruby/actions
[inch]: http://inch-ci.org/github/michaelherold/ksuid-ruby
[maintainability]: https://codeclimate.com/github/michaelherold/ksuid-ruby/maintainability
[test-coverage]: https://codeclimate.com/github/michaelherold/ksuid-ruby/test_coverage

ksuid is a Ruby library that can generate and parse [KSUIDs](https://github.com/segmentio/ksuid). The original readme for the Go version of KSUID does a great job of explaining what they are and how they should be used, so it is excerpted here.

---

# What is a KSUID?

KSUID is for K-Sortable Unique IDentifier. It's a way to generate globally unique IDs similar to RFC 4122 UUIDs, but contain a time component so they can be "roughly" sorted by time of creation. The remainder of the KSUID is randomly generated bytes.

# Why use KSUIDs?

Distributed systems often require unique IDs. There are numerous solutions out there for doing this, so why KSUID?

## 1. Sortable by Timestamp

Unlike the more common choice of UUIDv4, KSUIDs contain a timestamp component that allows them to be roughly sorted by generation time. This is obviously not a strong guarantee as it depends on wall clocks, but is still incredibly useful in practice.

## 2. No Coordination Required

[Snowflake IDs][1] and derivatives require coordination, which significantly increases the complexity of implementation and creates operations overhead. While RFC 4122 UUIDv1 does have a time component, there aren't enough bytes of randomness to provide strong protections against duplicate ID generation.

KSUIDs use 128-bits of pseudorandom data, which provides a 64-times larger number space than the 122-bits in the well-accepted RFC 4122 UUIDv4 standard. The additional timestamp component drives down the extremely rare chance of duplication to the point of near physical infeasibility, even assuming extreme clock skew (> 24-hours) that would cause other severe anomalies.

[1]: https://blog.twitter.com/2010/announcing-snowflake

## 3. Lexicographically Sortable, Portable Representations

The binary and string representations are lexicographically sortable, which allows them to be dropped into systems which do not natively support KSUIDs and retain their k-sortable characteristics.

The string representation is that it is base 62-encoded, so that they can "fit" anywhere alphanumeric strings are accepted.

# How do they work?

KSUIDs are 20-bytes: a 32-bit unsigned integer UTC timestamp and a 128-bit randomly generated payload. The timestamp uses big-endian encoding, to allow lexicographic sorting. The timestamp epoch is adjusted to March 5th, 2014, providing over 100 years of useful life starting at UNIX epoch + 14e8. The payload uses a cryptographically strong pseudorandom number generator.

The string representation is fixed at 27-characters encoded using a base 62 encoding that also sorts lexicographically.

---

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'ksuid'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install ksuid

## Usage

To generate a KSUID for the present time, use:

```ruby
ksuid = KSUID.new
```

If you need to parse a KSUID from a string that you received, use the conversion method:

```ruby
ksuid = KSUID.from_base62(base62_string)
```

If you need to interpret a series of bytes that you received, use the conversion method:

```ruby
ksuid = KSUID.from_bytes(bytes)
```

The `KSUID.from_bytes` method can take either a byte string or a byte array.

If you need to generate a KSUID for a specific timestamp, use:

```ruby
ksuid = KSUID.new(time: time)  # where time is a Time-like object
```

If you need to use a faster or more secure way of generating the random payloads (or if you want the payload to be non-random data), you can configure the gem for those use cases:

```ruby
KSUID.configure do |config|
  config.random_generator = -> { Random.new.bytes(16) }
end
```

### Prefixed KSUIDs

If you use KSUIDs in multiple contexts, you can prefix them to make them easily identifiable.

```ruby
ksuid = KSUID.prefixed('evt_')
```

Just like a normal KSUID, you can use a specific timestamp:

``` ruby
ksuid = KSUID.prefixed('evt_', time: time)  # where time is a Time-like object
```

You can also parse a prefixed KSUID from a string that you received:

```ruby
ksuid = KSUID::Prefixed.from_base62(base62_string, prefix: 'evt_')
```

Prefixed KSUIDs order themselves with non-prefixed KSUIDs as if their prefix did not exist. With other prefixed KSUIDs, they order first by their prefix, then their timestamp.

### ActiveRecord

Whether you are using ActiveRecord inside an existing project or in a new project, usage is simple. Additionally, you can use it with or without Rails.

_Note: In v1.0.0 of KSUID for Ruby, we will extract this behavior into a separate gem, `activerecord-ksuid`. It will be API-compatible with the implementation in v0.5.0+ of KSUID for Ruby. There will be upgrade instructions for v1.0.0 once we release it._

#### Adding to an existing model

Within a Rails project, it is very easy to get started using KSUIDs within your models. You can use the `ksuid` column type in a Rails migration to add a column to an existing model:

    rails generate migration add_ksuid_to_events ksuid:ksuid

This will generate a migration like the following:

```ruby
class AddKsuidToEvents < ActiveRecord::Migration[5.2]
  def change
    add_column :events, :unique_id, :ksuid
  end
end
```

Then, to add proper handling to the field, you will want to mix a module into the model:

```ruby
class Event < ApplicationRecord
  include ActiveRecord::KSUID[:unique_id]
end
```

#### Creating a new model

To create a new model with a `ksuid` field that is stored as a KSUID, use the `ksuid` column type. Using the Rails generators, this looks like:

    rails generate model Event my_field_name:ksuid

If you would like to add a KSUID to an existing model, you can do so with the following:

```ruby
class AddKsuidToEvents < ActiveRecord::Migration[5.2]
  change_table :events do |table|
    table.ksuid :my_field_name
  end
end
```

Once you have generated the table that you will use for your model, you will need to include a module into the model class, as follows:

```ruby
class Event < ApplicationRecord
  include ActiveRecord::KSUID[:my_field_name]
end
```

##### With a KSUID primary key

You can also use a KSUID as the primary key on a table, much like you can use a UUID in vanilla Rails. To do so requires a little more finagling than you can manage through the generators. When hand-writing the migration, it will look like this:

```ruby
class CreateEvents < ActiveRecord::Migration[5.2]
  create_table :events, id: false do |table|
    table.ksuid :id, primary_key: true
  end
end
```

You will need to mix in the module into your model as well:

```ruby
class Event < ApplicationRecord
  include ActiveRecord::KSUID[:id]
end
```

#### Outside of Rails

Outside of Rails, you cannot rely on the Railtie to load the appropriate files for you automatically. Toward the start of your application's boot process, you will want to require the following:

```ruby
require 'active_record/ksuid'

# If you will be using the ksuid column type in a migration
require 'active_record/ksuid/table_definition'
```

Once you have required the file(s) that you need, everything else will work as it does above.

#### Binary vs. String KSUIDs

These examples all store your identifier as a string-based KSUID. If you would like to use binary KSUIDs instead, use the `ksuid_binary` column type. Unless you need to be super-efficient with your database, we recommend using string-based KSUIDs because it makes looking at the data while in the database a little easier to understand.

When you include the KSUID module into your model, you will want to pass the `:binary` option as well:

```ruby
class Event < ApplicationRecord
  include ActiveRecord::KSUID[:my_field_name, binary: true]
end
```

#### Using a prefix on your KSUID field

For prefixed KSUIDs in ActiveRecord, you must pass the intended prefix during table definition so that the field is of appropriate size.

```ruby
class CreateEvents < ActiveRecord::Migration[5.2]
  create_table :events do |table|
    table.ksuid :ksuid, prefix: 'evt_'
  end
end
```

You also must pass it in the module builder that you include in your model:

```ruby
class Event < ApplicationRecord
  include ActiveRecord::KSUID[:ksuid, prefix: 'evt_']
end
```

You cannot use a prefix with a binary-encoded KSUID.

#### Use the KSUID as your `created_at` timestamp

Since KSUIDs include a timestamp as well, you can infer the `#created_at` timestamp from the KSUID. The module builder enables that option automatically with the `:created_at` option, like so:

```ruby
class Event < ApplicationRecord
  include ActiveRecord::KSUID[:my_field_name, created_at: true]
end
```

This allows you to be efficient in your database design if that is a constraint you need to satisfy.

## Contributing

So you’re interested in contributing to KSUID? Check out our [contributing guidelines](CONTRIBUTING.md) for more information on how to do that.

## Supported Ruby Versions

This library aims to support and is [tested against][actions] the following Ruby versions:

* Ruby 2.7
* Ruby 3.0
* Ruby 3.1
* JRuby 9.3

If something doesn't work on one of these versions, it's a bug.

This library may inadvertently work (or seem to work) on other Ruby versions, however support will only be provided for the versions listed above.

If you would like this library to support another Ruby version or implementation, you may volunteer to be a maintainer. Being a maintainer entails making sure all tests run and pass on that implementation. When something breaks on your implementation, you will be responsible for providing patches in a timely fashion. If critical issues for a particular implementation exist at the time of a major release, support for that Ruby version may be dropped.

## Versioning

This library aims to adhere to [Semantic Versioning 2.0.0][semver]. Violations of this scheme should be reported as bugs. Specifically, if a minor or patch version is released that breaks backward compatibility, that version should be immediately yanked and/or a new version should be immediately released that restores compatibility. Breaking changes to the public API will only be introduced with new major versions. As a result of this policy, you can (and should) specify a dependency on this gem using the [Pessimistic Version Constraint][pessimistic] with two digits of precision. For example:

    spec.add_dependency "ksuid", "~> 0.1"

[pessimistic]: http://guides.rubygems.org/patterns/#pessimistic-version-constraint
[semver]: http://semver.org/spec/v2.0.0.html

## License

The gem is available as open source under the terms of the [MIT License](https://opensource.org/licenses/MIT).
