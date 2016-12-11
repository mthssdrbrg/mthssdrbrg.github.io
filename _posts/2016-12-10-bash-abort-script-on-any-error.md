---
title: "bash: abort script on any error"
tags: bash
---

For the longest of time it's been my understanding that `set -e` in Bash causes a script to terminate if any single command of the script fails, which is also exactly what it does, in most cases.

However, when you start using functions things change a bit. Consider the following example:

```shell
$ cat basherr
#!/bin/bash

set -e
function run() {
  false # <- this should cause the script to fail
  echo ", world!"
}
output=$(run)
echo "Hello$output" # <- this should never be executed

$ basherr ; echo $?
Hello, world!
0
```

I was recently working on a script that had a couple of functions and I wanted the script to abort if any single command failed.
After some digging I found the `-E` option to `set`, which according to the documentation does the following:

> If set, any trap on ERR is inherited by shell functions, command substitutions, and commands executed in a subshell environment. The ERR trap is normally not inherited in such cases.

Armed with this newfound knowledge I replaced `set -e` with `set -E` and a custom trap handler for `ERR`:

```shell
set -E
trap "exit $?" ERR
```

and lo and behold:

```shell
$ basherr ; echo $?
1
```

I could have saved myself some headache by reading [the documentation](https://www.gnu.org/software/bash/manual/html_node/The-Set-Builtin.html) of `set` more thouroughly as it does outline exceptions where `set -e` won't do what one might think.
