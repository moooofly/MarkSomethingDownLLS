# Better Bash history

> 原文地址：https://sanctum.geek.nz/arabesque/better-bash-history/

By default, the Bash shell keeps the history of your most recent session in the `.bash_history` file, and the commands you’ve issued in your current session are also available with a `history` call. These defaults are useful for keeping track of what you’ve been up to in the shell on any given machine, but with disks much larger and faster than they were when Bash was designed, **a little tweaking in your `.bashrc` file can record history more permanently, consistently, and usefully**.

## Append history instead of rewriting it

You should start by setting the `histappend` option, which will mean that when you close a session, your history will be appended to the `.bash_history` file rather than overwriting what’s in there.

```
shopt -s histappend
```

## Allow a larger history file

The default maximum number of commands saved into the `.bash_history` file is a rather meager 500. If you want to keep history further back than a few weeks or so, you may as well bump this up by explicitly setting `$HISTSIZE` to a much larger number in your `.bashrc`. We can do the same thing with the `$HISTFILESIZE` variable.

```
HISTFILESIZE=1000000
HISTSIZE=1000000
```

The man page for Bash says that `HISTFILESIZE` can be `unset` to stop truncation entirely, but unfortunately this **doesn’t work** in `.bashrc` files **due to the order in which variables are set**; it’s therefore more straightforward to simply set it to a very large number.

If you’re on **a machine with resource constraints**, it might be a good idea to **occasionally archive old `.bash_history` files to speed up login and reduce memory footprint**.

## Don’t store specific lines

You can prevent commands that start with a space from going into history by setting `$HISTCONTROL` to `ignorespace`. You can also ignore duplicate commands, for example repeated `du` calls to watch a file grow, by adding `ignoredups`. There’s a shorthand to set both in `ignoreboth`.

```
HISTCONTROL=ignoreboth
```

You might also want to remove the use of certain commands from your history, whether for privacy or readability reasons. This can be done with the `$HISTIGNORE` variable. It’s common to use this to exclude `ls` calls, **job control** builtins like `bg` and `fg`, and calls to `history` itself:

```
HISTIGNORE='ls:bg:fg:history'
```

## Record timestamps

If you set `$HISTTIMEFORMAT` to something useful, Bash will record the timestamp of each command in its history. In this variable you can specify the format in which you want this timestamp displayed when viewed with `history`. I find the full date and time to be useful, because it can be sorted easily and works well with tools like `cut` and `awk`.

```
HISTTIMEFORMAT='%F %T '
```

## Use one command per line

To make your `.bash_history` file a little easier to parse, you can force commands that you entered on more than one line to be adjusted to fit on only one with the `cmdhist` option:

```
shopt -s cmdhist
```

## Store history immediately

**By default, Bash only records a session to the `.bash_history` file on disk when the session terminates. This means that if you crash or your session terminates improperly, you lose the history up to that point**. You can fix this by recording each line of history as you issue it, through the `$PROMPT_COMMAND` variable:

```
PROMPT_COMMAND='history -a'
```

## Related Posts

- [Bash history expansion](https://sanctum.geek.nz/arabesque/bash-history-expansion/)
- [Bash prompts](https://sanctum.geek.nz/arabesque/bash-prompts/)
- [Shell config subfiles](https://sanctum.geek.nz/arabesque/shell-config-subfiles/)


