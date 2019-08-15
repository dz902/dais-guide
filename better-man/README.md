# Dai's Better Man Pages

## Basics

* __`STDIN` / `STDOUT` vs args.__

## `xargs`

_"Split `STDIN` into args and execute a command with args."_

* __Use CLI to get a few instances and terminate them.__ CLI output is in `STDIN`, you need to read them in `STDIN`, split them into args separated by spaces, then call the termination command.

### See Also

* __`tee`__. Use this command if you want to output something to `STDOUT` with `xargs`, because `>` would be sent to `xargs` as args.