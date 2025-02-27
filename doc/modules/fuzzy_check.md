---
layout: doc
title: Fuzzy check module
---
# Fuzzy check module

This module is intended to check messages for specific fuzzy patterns stored in
[fuzzy storage workers](../workers/fuzzy_storage.html). At the same time, this module
is responsible for teaching fuzzy storage with message patterns.

## Fuzzy patterns

Rspamd uses the `shingles` algorithm to perform a fuzzy match of messages. This algorithm
is probabilistic and uses word chains to detect some common patterns, and thus filter
as spam or ham messages. The `shingles` algorithm is described in the following 
[research paper](http://dl.acm.org/citation.cfm?id=283370). We use 3-gramms for this
algorithm and a set of hash functions: siphash, mumhash and others. Currently,
rspamd uses 32 hashes for shingles.

Attachments and images are not currently matched against fuzzy hashes, but they
are checked by way of blake2 digests using strict match.

## Module configuration

The fuzzy check module has several global options and allows to specify multiple match
storages. Global options include:

- `symbol`: default symbol to insert (if no flags match)
- `min_length`: minimum length of text parts in words to perform fuzzy check (default - check all text parts)
- `min_bytes`: minimum length of attachments and images in bytes to check them in fuzzy storage
- `whitelist`: IPs in this list bypass all fuzzy checks
- `timeout`: timeout to wait for a reply

Fuzzy rules are defined as a set of `rule` definitions. Each `rule` must have a `servers`
list to check or learn, and a set of flags and optional parameters. Here is an example:

~~~ucl
# local.d/fuzzy_check.conf
rule "FUZZY_CUSTOM" {
  # List of servers. Can be an array or multi-value item
  servers = "127.0.0.1:11335";

  # List of additional mime types to be checked in this fuzzy ("*" for any)
  mime_types = ["application/*", "*/octet-stream"];

  # Maximum global score for all maps
  max_score = 20.0;

  # Ignore flags that are not listed in maps for this rule
  skip_unknown = yes;

  # If this value is false (i.e. no), then allow learning for this fuzzy rule
  read_only = no;

  # Fast hash type
  algorithm = "mumhash";
}
~~~

Each rule can have several `fuzzy_map` values, ordered by an ordinal `flag` value. A single
fuzzy storage can contain both good and bad hashes that should have different symbols,
and thus, different weights. Maps are defined inside fuzzy rules as follows:

~~~ucl
# local.d/fuzzy_check.conf
rule "FUZZY_LOCAL" {
...
fuzzy_map = {
  FUZZY_DENIED {
    # Maximum weight for this list
    max_score = 20.0;
    # Flag value
    flag = 1
  }
  FUZZY_PROB {
    max_score = 10.0;
    flag = 2
  }
  FUZZY_WHITE {
    max_score = 2.0;
    flag = 3
  }
}
~~~

The meaning of `max_score` can be rather unclear. First of all, all hashes in
fuzzy storage have their own weights. For example, if we have a hash `A` and 100 users
marked it as a spam hash, then it will have a weight of `100 * single_vote_weight`.
Therefore, if a `single_vote_weight` is `1` then the final weight will be `100`.
`max_score` is the weight that is required for the rule to add symbol with the maximum
score 1.0 (that will be of course multiplied by the `metric` value's weight).
For example, if the `weight` of the hash is `100` and the `max_score` is set to `20`,
then the rule will be added with the weight of `1`. If `max_score` is set to `200`,
then the rule will be added with the weight likely `0.2` (calculated via hyperbolic tangent).
In the following configuration:

~~~ucl
metric {
	name = "default";
	...
	symbol {
		name = "FUZZY_DENIED";
		weight = "10.0";
	}
	...
}
fuzzy_check {
	rule {
	...
	fuzzy_map = {
		FUZZY_DENIED {
			# Maximum weight for this list
			max_score = 20.0;
			# Flag value
			flag = 1
        }
        ...
    }
}
~~~

If a hash has value `10`, then a symbol `FUZZY_DENIED` with weight of `2.0` will be added.
If a hash has value `100500`, then `FUZZY_DENIED` will have weight `10.0`.

## Training fuzzy_check

Module `fuzzy_check` can also learn from messages. You can use `rspamc` command or
connect to the **controller** worker using HTTP protocol. For learning, you must check 
the following settings:

1. Controller worker should be accessible by `rspamc` or HTTP (check `bind_socket`)
2. Controller should allow privilleged commands for this client (check `enable_password` or `allow_ip` settings)
3. Controller should have `fuzzy_check` module configured to the servers specified
4. You should know `fuzzy_key` and `fuzzy_shingles_key` to operate with this storage
5. Your `fuzzy_check` module should have `fuzzy_map` configured to the flags used by server
6. Your `fuzzy_check` rule must have `read_only` option turned off (`read_only = false`)
7. Your `fuzzy_storage` worker should allow updates from the controller's host (`allow_update` option)
8. Your controller should be able to communicate with fuzzy storage by means of the `UDP` protocol

If all these conditions are met, then you can teach rspamd messages with rspamc:

	rspamc -w <weight> -f <flag> fuzzy_add ...

or delete hashes:

	rspamc -f <flag> fuzzy_del ...

you can also delete a hash that you find in the log output:

	rspamc -f <flag> fuzzy_delhash <hash-id>

On learning, rspamd sends commands to **all** servers inside a specific rule. On check,
rspamd selects a server in a round-robin manner.

## Usage of the feeds provided by `rspamd.com`

If you use `rspamd.com` feeds (enabled by default) you need to qualify [**free usage policy**](https://rspamd.com/doc/usage_policy.html) or you would be blocked from using this service. There is a special symbol called `FUZZY_BLOCKED` that means that you violate these terms and are no longer permitted to use this service. This symbol has no weight and it should not affect any mail processing operations.
