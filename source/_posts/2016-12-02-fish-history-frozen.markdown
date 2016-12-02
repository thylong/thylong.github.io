---
layout: post
title:  "Debug frozen Fish shell history"
description: "A reminder on how $PATH behavior"
date:   2016-12-02 09:00:00
keywords: "path, env, fish, fishshell, golang, conflict, bash, zsh"
category: debug
comments: true
---

<h2>Debug frozen Fish shell history</h2>

If you're reading that article right now, I assume you're already using fish as I do.
This is really a great tool despite that it sometimes misses autocompletions. I feel more productive since I started to use it as my default shell.

If you don't use it yet, I'm not saying you should because it comes with good and bad sides.
Check it out [here][fish_official].

It always worked great until one day, when I figured out that whenever I was using the `history` command, the prompt was freezing indefinitely.

{% highlight bash %}
# Examples of freezing commands
history | grep 'mongo'
history search 'mongo'
{% endhighlight %}

This is really painful as you cannot always remember what you typed to connect to a database, what was the last package you installed using a particular package manager or whatever.

<h3>Debugging Fish's behavior</h3>

Thankfully, Fish comes with a set of handy debugging options.
You can execute a command and increase the verbosity of the output.
With the history command it becomes :

{% highlight bash %}
# o -c or --command=COMMANDS evaluate the specified commands instead of reading from the commandline
# o -d or --debug-level=DEBUG_LEVEL specify the verbosity level of fish. A higher number means higher verbosity. The default level is 1.
fish -d 4 -c history
{% endhighlight %}

The output must look something like this :

{% highlight bash %}
...
<4> fish: io_buffer_t::read: blocking read on fd 3
<4> fish: Exec job 'isatty stdout' with id 2
<4> fish: Exec job 'set -l fd 0' with id 3
<3> fish: Skipping fork: no output for internal builtin 'set'
<3> fish: Set status of set -l fd 0 to 0 using short circuit
<3> fish: Job is constructed
<4> fish: Continue job 3, gid 0 (set -l fd 0), COMPLETED, NON-INTERACTIVE
<4> fish: Exec job 'count $argv >/dev/null' with id 3
<2> fish: Fork #56, pid 91785: internal builtin for 'count'
<3> fish: Job is constructed
<4> fish: Continue job 3, gid 91658 (count $argv >/dev/null), UNCOMPLETED, NON-INTERACTIVE
<4> fish: Exec job 'set fd 1' with id 3
<3> fish: Skipping fork: no output for internal builtin 'set'
<3> fish: Set status of set fd 1 to 0 using short circuit
<3> fish: Job is constructed
<4> fish: Continue job 3, gid 0 (set fd 1), COMPLETED, NON-INTERACTIVE
<3> fish: path_get_path( 'test' )
<4> fish: Exec job 'command test -t "$fd"' with id 3
<4> fish: env_export_arr() recalc
<2> fish: Fork #57, pid 91786: spawn external command '/Users/hello/www/repositories/bin/test' from '<no file>'
<3> fish: Job is constructed
<4> fish: Continue job 3, gid 91658 (command test -t "$fd"), UNCOMPLETED, NON-INTERACTIVE
{% endhighlight %}

We can see in this case that the blocking subroutine is a command executing the program called `test`, located in `/Users/hello/www/repositories/bin/test`.
If I reset the `$PATH` command to its original value, the test command point to `/bin/test`

<h3>Looking for an explanation</h3>

Here is the content of my `$PATH` environment variable :

{% highlight bash %}
[hello@wks-156 ~]$ echo $PATH
/Users/hello/www/repositories/bin /usr/local/sbin /opt/google-cloud-sdk/bin /usr/local/bin /usr/bin /bin /usr/sbin /sbin /opt/X11/bin /usr/local/go/bin /usr/local/MacGPG2/bin /usr/local/bin /Users/hello/anaconda/bin
{% endhighlight %}

`/Users/hello/www/repositories/bin` is pointing to my `$GOBIN` directory. Meaning it contains all the built GO executables. The `go test` is one of them and is present as `$PWD/test`.

And If we read the PATH environment variable official definition :

> This variable shall represent the sequence of path prefixes that certain functions and utilities apply in searching for an executable file known only by a filename. The prefixes shall be separated by a colon ( ':' ). When a non-zero-length prefix is applied to this filename, a slash shall be inserted between the prefix and the filename. A zero-length prefix is a legacy feature that indicates the current working directory. It appears as two adjacent colons ( "::" ), as an initial colon preceding the rest of the list, or as a trailing colon following the rest of the list. A strictly conforming application shall use an actual pathname (such as .) to represent the current working directory in PATH . `The list shall be searched from beginning to end`, applying the filename to each prefix, until an executable file with the specified name and appropriate execution permissions is found. If the pathname being sought contains a slash, the search through the path prefixes shall not be performed. If the pathname begins with a slash, the specified path is resolved (see Pathname Resolution). If PATH is unset or is set to null, the path search is implementation-defined.

It's now clear.
The first command found in the PATH directories is the one that will be triggered by the shell (Fish).
In my case, the go programs were prioritized over the built in programs.

To summarize, any modification to PATH, CDPATH, GOPATH or other "path like" variable should be applied like this :
{% highlight bash %}
set -gx PATH $PATH /Users/hello/www/repositories/bin
{% endhighlight %}

A good slap in the face to remember the usage of this variable.
I hope that I will help some of you !


[fish_official]:  http://fishshell.com
