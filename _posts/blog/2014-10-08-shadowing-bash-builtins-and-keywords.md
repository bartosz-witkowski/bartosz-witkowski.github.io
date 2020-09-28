---
layout: post
title: Shadowing bash builtins and keywords
permalink: /2014/10/08/shadowing-bash-bulitins-and-keywords.html
categories:
- blog
---

Somethings that trip me up from time to time are bash bultins and bash keywords
that shadow normal commands. The list isn't long and you don't hit the problem
that often but when you do... it's kind of a wild ride of questioning your
sanity until you remember to check `type $COMMAND`, the command `which`
obviously doesn't support bash builtin lookup.

You can enforce to use the command instead of the shell keyword by prefixing `\`
before the command or using `command $COMMAND` builtin (note that shell builtins
!= shell keywords).

The list of bash builtin and keywords that shadow normal commands is as follows:

* `[` &nbsp;
* `echo` 
* `false` 
* `kill` 
* `printf` 
* `pwd` 
* `test` 
* `time` 
* `true`     

I created it using <code>for b in `ls $P` ; do type $b ; done | grep -v $P</code> and
setting a `P` variable to `/bin/` and `/usr/bin`

Some differences in relation to the GNU implementations (not counting the
obvious ones like the builtins not spawning a process etc).

All
----

Bultin commands don't recognize the `--version` and `--help` flags.  This won't
b repeated in the list below

`[`/`test`
----------

`/usr/bin/[` doesn't support:

* `-N` file
* `-o` optname
* `-v` varname


`echo`
------

Using version `4.2.37` that comes with my debian this properly supports `-e` and
`-E` flags, but some of the previous versions had problems with them.

`kill`
------

The builtin doesn't support the following options:
* `-L`/`--table` 
* `-s`/`--signal` options (though they're exactly the same as the `-` option)

The builtin has different output for the `-l` option and doesn't support the
long option `--list`.

`printf`
--------

The builtin doesn't recognize the `\c` option.

`pwd`
-----

The builtin doesn't support the long options `--logical` and `--physical`.

`time`
------

The builtin has different output format compare:

	$ time sleep 0

	real	0m0.001s
	user	0m0.004s
	sys	0m0.000s
	$ /usr/bin/time sleep 0
	0.00user 0.00system 0:00.00elapsed ?%CPU (0avgtext+0avgdata 604maxresident)k
	0inputs+0outputs (0major+197minor)pagefaults 0swaps

The builting doesn't support the following options:
* `-o`/`--output`
* `-a`/`--append` 
* `-f`/`--format` 
* `-v`/`--verbose` 
* `--portability` - though the short option `p` is supported
* `--quiet`
* `--V` 
