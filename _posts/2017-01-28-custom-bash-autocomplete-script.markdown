---
layout: post
title: Custom bash autocomplete script
description: A walkthrough in building a bash autocompletion script
date: 2017-01-28
tags:
    - bash
    - autocomplete
    - autocompletion
---


## Introduction

At my company, we wrote a simple bash script to manage our project toolchain. I will name this command `projectctl`.

As it is quite convenient, I wanted to provide autocompletion in order to ease developing. I will explain my walk through this *journey* ;)

## Search and study

First, I searched the internet for some resources. These were the one I found useful :

* [An Introduction to Programmable Completion](http://tldp.org/LDP/abs/html/tabexpansion.html)
* [pyenv bash autocomplete](https://github.com/yyuu/pyenv/blob/master/completions/pyenv.bash)
* [Programmable Completion Builtins](https://www.gnu.org/software/bash/manual/html_node/Programmable-Completion-Builtins.html)

But it is not enough to just do a copy paste. What is important is to study and understand what you are doing.

Basically, autocompletion in bash is provided by one builtin command `complete`. It is quite simple to use. You indicate to bash what to provide as argument for a command.

For example :

```shell
$ complete -W "action1 action2" projectctl
$ projectctl [Tab]
# The first tab will expend the argument to
$ projectctl action[Tab]
action1 action2
```

Here we are just providing a list of words to expend and use as possible argument thanks to the `-W` argument

Before jumping to the next section, let's talk a little about `compgen`. This builtin command is not really useful on its own but it will become important once we implement more complicated completion rules.

It is simple, it supports almost the same options as `complete` and can take one optional parameters, the word to match with the expended values from the options. It returns an expended list of words matching the provided word and writes it to stdout. But let's look at a small example to understand :

```shell
$ compgen -W "arg1 pos1 opt1 arg2"
arg1
pos1
opt1
arg2
$ compgen -W "arg1 pos1 opt1 arg2" ar
arg1
arg2
```

Now we have only seen how to autocomplete a simple example, but what if we have something more complex. Let's study this with our `projectctl` command.

## Our project CLI tool

We will use a simple script for testing purposes :

```shell
$ cat projectctl
#!/bin/bash

echo "command : $0"
echo "arguments : $@"
```

Let's execute it :

```shell
$ projectctl arg1 arg2 -d
command : projectctl
arguments : arg1 arg2 -d
```

The rules our `projectctl` parameters and options must fulfill are :

* The full command is : `projectctl [-v] <service> [<action>] [-d] [-q]`
* optional `-v` option authorized as the first parameter only
* Optional first parameter `service` in the list : `service1`, `service2`, `help`
* Optional second parameter `action` :
    * only available if we have a service just before.
    * None if `service` is `help`.
    * Else `start`, `stop`, `reload` or `help` if action is `service1`
    * Else `start`, `stop` or `help` if action is `service2`
* Optional options only at the end of the line : `-d` or `-q` if action is not `help`

We can understand quickly that a simple `complete` configuration won't be enough anymore. Fortunately, `complete` provides a way to execute a function to decide the available words for autocompletion thanks to its `-F` option. When bash executes this function, it initiates some environment variable to understand where we stand in the process of writing the command in order. The goal of the function is to fill the `COMPREPLY` variable with an array of words to provide as autocompletion candidates.

```shell
$ cat projectctl.bash
_projectctl() {
  COMPREPLY=()

  echo ""
  echo "COMP_WORDS : ${COMP_WORDS}"
  echo "COMP_CWORD : ${COMP_CWORD}"
  echo "COMP_WORDS[COMP_CWORD] : ${COMP_WORDS[COMP_CWORD]}"
  echo "COMP_LINE : ${COMP_LINE}"
  echo "COMP_POINT : ${COMP_POINT}"
  echo "COMP_KEY : ${COMP_KEY}"
  echo "COMP_TYPE : ${COMP_TYPE}"
  echo "args : $@"
  echo "reply : ${COMPREPLY}"
}

complete -F _projectctl projectctl

$ source projectctl.bash

$ projectctl [Tab]
COMP_WORDS : projectctl
COMP_CWORD : 1
COMP_WORDS[COMP_CWORD] :
COMP_LINE : projectctl
COMP_POINT : 13
COMP_KEY : 9
COMP_TYPE : 9
args : projectctl  projectctl
reply :

$ projectctl acti[Tab]
COMP_WORDS : projectctl
COMP_CWORD : 1
COMP_WORDS[COMP_CWORD] : acti
COMP_LINE : projectctl acti
COMP_POINT : 16
COMP_KEY : 9
COMP_TYPE : 9
args : projectctl acti projectctl
reply :
```

We can easily understand the information each variable is providing :

* `COMP_WORDS` : an array variable consisting of the individual words in the current command line split automatically.
* `COMP_CWORD` : an index into `${COMP_WORDS}` of the word containing the current cursor position. `COMP_WORDS` is zero indexed so in the first tentative `${COMP_WORDS[COMP_CWORD]}` was empty and in the second its value was `acti`
* `COMP_LINE` : the current command line
* `COMP_POINT` : the index of the current cursor position relative to the beginning of the current command
* `COMP_KEY` and `COMP_TYPE` provides an integer to get information about the pressed key which triggered the autocompletion and the type of autocompletion. I won't enter into these details as it is quite advanced and I did not study it as I did not need it. We will focus on standard command line autocompletion with Tab key only.
* `$1` : the first argument is the name of the command whose arguments are being completed
* `$2` : the second argument is the word being completed (empty in the first try as we have not yet typed any character)
* `$3` : the word preceding the word being completed on the current command line

And we have one additional important variable :

* **`COMPREPLY`** : this is an array of string which must contain all available values for autocompletion. This is what you need to fill when you implement this autocompletion function.

So let's look at a working example :

```shell
_projectctl() {
  COMPREPLY=()

  # All possible first values in command line
  local SERVICES=("-v" "service1" "service2")

  # declare an associative array for options
  declare -A ACTIONS
  ACTIONS[service1]="help reload start stop"
  ACTIONS[service2]="help start stop"

  # All possible options at the end of the line
  local OPTIONS=("-d" "-q")

  # current word being autocompleted
  local cur=${COMP_WORDS[COMP_CWORD]}

  # If previous arg is -v it means that we remove -v from SERVICES for autocompletion
  if [ $3 = "-v" ] ; then
    SERVICES=${SERVICES[@]:1}
  fi

  # If previous arg is a key of ACTIONS (so it is a service).
  # It means that we must display action choices
  if [ ${ACTIONS[$3]+1} ] ; then
    COMPREPLY=( `compgen -W "${ACTIONS[$3]}" -- $cur` )
  # If previous arg is one of the actions or previous arg is an option
  # We are at the end of the command and only options are available
  elif [[ "${ACTIONS[*]}" == *"$3"* ]] || [[ "${OPTIONS[*]}" == *"$3"*  ]]; then
    # SPecial use case : help does not support options
    if [ "$3" != "help" ] ; then
      COMPREPLY=( `compgen -W "${OPTIONS[*]}" -- $cur` )
    fi
  else
    # if everything else does not match, we are either :
    # - first arg waiting for -v or a service code
    # - second arg with first being -v. waiting for a service code.
    COMPREPLY=( `compgen -W "${SERVICES[*]}" -- $cur` )
  fi
}

complete -F _projectctl projectctl
```

And running :

```shell
$ projectctl
service1  service2  -v
$ projectctl -v service
service1  service2
$ projectctl service
service1  service2
$ projectctl service1
help    reload  start   stop
$ projectctl service2
help   start  stop
$ projectctl service2 help
$ projectctl service2 start -
-d  -q
$ projectctl service2 start -d -
-d  -q
```

**We have successfully implemented the rules we decided upon at the start.** There are some useful things to remember about our bash script :

* Bash supports associative array :

```shell
declare -A ACTIONS
ACTIONS[service1]="help reload start stop"
ACTIONS[service2]="help start stop"
```

* And of course classic array :

```shell
SERVICES=("-v" "service1" "service2")
```

*Note : only one dimensional array are supported so you cannot have an array in an array*

* We can declare `local` variable to a function :

```shell
local cur=${COMP_WORDS[COMP_CWORD]}
```

*Note that `declare` is always `local` to the block scope*

* We can remove the first N items of an array :

```shell
SERVICES=${SERVICES[@]:N}
```

`@` means that each element of the array will be processed as a different shell-word (list expansion) and we start at the Nth element.

* Check that an associative array has a key :

```shell
[ ${ACTIONS[$3]+1} ]
```

An example explains everything :

```shell
$ declare -A array
$ array=([key1]=toto [key2]=tata)
$ [ ${array[key1]+1} ]
$ echo $?
0
$ [ ${array[key3]+1} ]
$ echo $?
1
```

*Note that the `+1` could have been anything (`+qsqdq`). It is used to evaluate the expression to true if the key is found and the value associated to it is falsy*

* Check that a string contains another string (using `*` glob selector):

```shell
$ mystring="start stop help"
$ [[ "$mystring" == *"stop"* ]]
$ echo $?
0
$ [[ "$mystring" == *"reload"* ]]
$ echo $?
1
```

* The difference between `[*]` and `[@]` on a bash array :

```shell
$ array=("start" "stop" "help")
$ echo "${array[*]}"
start stop help
# equivalent to "start stop help"
$ echo "${array[@]}"
start stop help
# equivalent to "start" "stop" "help"
```

* Debug an array (useful to print its content):

```shell
$ declare -p ACTIONS
declare -A ACTIONS='([service2]="help start stop" [service1]="help start stop reload" )'
```

Now what can we improve !!! Instead of having a static list of services and actions in our script, it would be great that the autocompletion script updates itself automatically when we add new services or actions to the main script. To do that, we can follow [pyenv example](https://github.com/yyuu/pyenv/blob/master/completions/pyenv.bash). The `projectctl` command will provide an *hidden* `commands` action which will be used to get authorized values in the autocomplete script.

To provide a working example, I will update the projectctl script this way :

```shell
#!/bin/bash

if [ "$1" == "commands" ] ; then
  echo "service1 service2"
fi

if [ "$1" == "service1" ] && [ "$2" == "commands" ] ; then
  echo "help start stop reload"
fi

if [ "$1" == "service2" ] && [ "$2" == "commands" ] ; then
  echo "help start stop"
fi
```

Let's execute it to see the result :

```shell
$ projectctl commands
service1 service2
$ projectctl service1 commands
help start stop reload
$ projectctl service2 commands
help start stop
```

Based on the previous version of our complete script, we will need to automatically fill the `SERVICES` with the result of `projectctl commands` and for each service, we must create an entry in the array `ACTIONS` filled with the result of `projectctl <service> commands`. So we just need to refactor the top of our script.

This is a working solution :

```shell
  COMPREPLY=()

  # Creates an array with the list of all services
  local SERVICES=( `projectctl commands` )

  # Creates the mapping service => string with all authorized actions separated by a space
  declare -A ACTIONS
  for i in "${SERVICES[@]}"
  do
    ACTIONS[$i]=`projectctl $i commands`
  done

  # Prepends the -v option which is authorized as a first args
  SERVICES=("-v" "${SERVICES[@]}")

  # All possible options at the end of the line
  local OPTIONS=("-d" "-q")

  # The script does not change after this
  ...
```

And it works great :

```shell
$ projectctl [Tab]
service1  service2  -v
$ projectctl -v service[Tab]
service1  service2
$ projectctl -v service1 st[Tab]
start  stop
$ projectctl -v service1 [Tab]
help    reload  start   stop
$ projectctl service2 [Tab]
help   start  stop
$ projectctl service2 stop -[Tab]
-d  -q
```

So what did we learn here :

* to split words into an array (string expansion):

```shell
$ SERVICES="service1 service2"
$ IFS=' ' array=( $SERVICES )
$ declare -p array
declare -a array='([0]="service1" [1]="service2")'
```

`$IFS` is the internal field separator. This variable determines how Bash recognizes fields, or word boundaries, when it interprets character strings. Here we update temporary IFS for the string expansion operation. Note that it is useless to configure it in our example as space is already one of the default separator.

```shell
$ SERVICES="service1|service2"
$ IFS='|' array=( $SERVICES )
$ declare -p array
declare -a array='([0]="service1" [1]="service2")'
```

However beware of this method which is sensible to path expansion :

```shell
$ ls
projectctl projectctl.bash
$ SERVICES="projectct[lad] service2"
$ IFS=' ' array=( $SERVICES )
$ declare -p array
declare -a array='([0]="projectctl" [1]="service2")'
```

`projectct[lad]` has been expended to a file found in the current working directory `projectctl`. Read more about path expansion here : [Filename Expansion](https://www.gnu.org/software/bash/manual/html_node/Filename-Expansion.html) or here [Pathname expansion](http://wiki.bash-hackers.org/syntax/expansion/globs)

One other method I found on [stackoverflow](http://stackoverflow.com/questions/10586153/split-string-into-an-array-in-bash) was to use the `read` builtin.

```shell
$ SERVICES="service1|service2"
$ IFS='|' read -r -a array <<< "$SERVICES"
$ declare -p array
declare -a array='([0]="service1" [1]="service2")'
```

This single line of code is full of things :

* `<<<` : the *here string*. The official document says that the [here string](https://linux.die.net/abs-guide/x15683.html) can be considered as a stripped down version of the [here document](https://linux.die.net/abs-guide/here-docs.html).

Let's focus on the `here document` first. An example speaks always better on its own (taken from the above mentioned link):

```shell
ftp -n $Server <<End-Of-Session
user anonymous "$Password"
binary
bell
cd $Directory
put "$Filename.lsm"
put "$Filename.tar.gz"
bye
End-Of-Session
```

It allows us to feed a command list to an **interactive** program or command.

Now for the `here string`, it consists of nothing more than `COMMAND <<< $WORD`, where `$WORD` is expanded and fed to the stdin of `COMMAND`.

So `read -r -a array <<< "$SERVICES"` expands `$SERVICES` and fed it to the `read` builtin. The parameter `-r` disables backslash escapes and line-continuation in the read data and `-a` will put what is received from stdin in the `$array` variable.

This attempt to make an autocompletion settings for my project tool chain allowed me to discover some nice tips in bash programming. I am not an expert but I feel I benefited a lot of things from this work. Don't hesitate to comment on this article. I probably won't be able to help you if you are faced with a difficult bash programming problem as I am not an expert but I will do my best.

And now the final version of the autocompletion script :

```shell
_projectctl() {
  COMPREPLY=()

  # Creates an array with the list of all services
  local SERVICES=( `projectctl commands` )

  # Creates the mapping service => string with all authorized actions separated by a space
  declare -A ACTIONS
  for i in "${SERVICES[@]}"
  do
    ACTIONS[$i]=`projectctl $i commands`
  done

  # Prepends the -v option which is authorized as a first args
  SERVICES=("-v" "${SERVICES[@]}")

  # All possible options at the end of the line
  local OPTIONS=("-d" "-q")

  # current word being autocompleted
  local cur=${COMP_WORDS[COMP_CWORD]}

  # If previous arg is -v it means that we remove -v from SERVICES for autocompletion
  if [ $3 = "-v" ] ; then
    SERVICES=${SERVICES[@]:1}
  fi

  # If previous arg is a key of ACTIONS (so it is a service).
  # It means that we must display action choices
  if [ ${ACTIONS[$3]+1} ] ; then
    COMPREPLY=( `compgen -W "${ACTIONS[$3]}" -- $cur` )
  # # If previous arg is one of the actions or previous arg is an option
  # We are at the end of the command and only options are available
  elif [[ "${ACTIONS[*]}" == *"$3"* ]] || [[ "${OPTIONS[*]}" == *"$3"*  ]]; then
    # SPecial use case : help does not support options
    if [ "$3" != "help" ] ; then
      COMPREPLY=( `compgen -W "${OPTIONS[*]}" -- $cur` )
    fi
  else
    # if everything else does not match, we are either :
    # - first arg waiting for -v or a service code
    # - second arg with first being -v. waiting for a service code.
    COMPREPLY=( `compgen -W "${SERVICES[*]}" -- $cur` )
  fi
}

complete -F _projectctl projectctl
```