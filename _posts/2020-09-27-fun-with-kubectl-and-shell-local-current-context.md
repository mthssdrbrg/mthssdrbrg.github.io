---
title: fun with kubectl and local current-context
tags: kubernetes kubectl current-context shell fish oddity
date: 2020-09-27 13:55 +0100
---

I use [`kubectl`](https://kubernetes.io/docs/reference/kubectl/overview/) on a daily basis and it's a _great_ tool.
However, something that has always bothered me is that setting a current context updates its configuration file and thus
for _all_ terminal sessions. This can be surprising if you're like me and switch between different contexts and terminal
sessions as running a command in shell A, then switching context in shell B and running a command, the context in shell A
will have changed!

I've previously looked at something like [`kubie`](https://github.com/sbstp/kubie) but it doesn't (currently) have
support for `fish` (yet), and my [Rust](https://www.rust-lang.org/) is pretty bad at the moment, although adding support
for fish is probably a good way to _improve_ that.

One possible solution would be to use multiple different configuration files, and set the `KUBECONFIG` environment
variable accordingly. My main issue with that is that credentials (such as refresh token(s) from OIDC/OAuth2) are
automatically refreshed and persisted to the configuration file, thus rendering all of the other configuration files out
of date and using a different one (eventually) causes another refresh and so on and so forth.

A simple trick that I've found to work around this is to have a "main" configuration file that holds clusters, contexts,
users, etc but that does not set the `current-context`, and then have multiple "context" configuration files that _only_
set the `current-context` and that are _read-only_.  This has the nice side-effect that it's no longer possible to set
the `current-context` (as shown below), _but_ updated credentials is persisted to the main configuration file
automatically!

```shell
$ kubectl config current-context
test5
$ kubectl config set current-context test6
error: open /home/dist/.config/kubectx/test5: permission denied
```

Whether this is intended behaviour or not I'm not sure, but it works wonderfully well. I've combined this into a `fish`
function which also overrides `Ctrl+D` to "exit" from the current context:

```shell
# kubectx.fish
function kubectx --argument-names context
  if test -n $context
    set --local kubectx_config $XDG_CONFIG_HOME/kubectx/$context

    set --global --export KUBECTX $context
    set --global --export KUBECONFIG $kubectx_config:$HOME/.kube/config

    if not test -f $kubectx_config
      sed -e "s/%CONTEXT%/$context/g" $XDG_CONFIG_HOME/kubectx/template > $kubectx_config
      chmod 400 $kubectx_config
    end

    bind -M insert \cd __kubectx_exit
    bind \cd __kubexit_exit
    bind -M visual \cd __kubexit_exit

    commandline --function repaint
  else
    echo "<context> must not be empty" >&2
    return 1
  end
end

function __kubectx_exit
  set --erase KUBECTX
  set --erase KUBECONFIG
  bind --erase -M insert \cd
  bind --erase \cd
  bind --erase -M visual \cd
  commandline --function repaint
end

# $XDG_CONFIG_HOME/kubectx/template
apiVersion: v1
kind: Config
current-context: "%CONTEXT%"
```

I then use this together with [fzf](https://github.com/junegunn/fzf) to more easily switch between contexts:

```shell
# __k8s_cluster_search.fish
function __k8s_cluster_search --description 'k8s cluster search'
  set --local selected (kubectl config get-contexts --output name | eval (__fzfcmd) $FZF_DEFAULT_OPTS)
  if string length -q -- $selected
    kubectx $selected
  end
end
```

Latest version of the above can also be found in my [dotfiles
repository](https://github.com/mthssdrbrg/dotfiles).
