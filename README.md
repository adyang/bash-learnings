# Bash Scripting: The Safe Way

## Contents
1. Introduction
2. Always Quote
3. Use Arrays
4. Conditionals
5. Handle Exit Status
6. I/O & Redirection
7. Tools
8. References

## Introduction
### When to use Bash
Bash is best used as a tool to handling arguments and programs. E.g. wrapper start/ stop scripts.
 
### The First Line
* All bash scripts should start with either `#!/bin/bash` or  `#!/usr/bin/env bash`
* This line will inform the kernel of what executable should be used to interpret the script
* `#!/bin/bash` is preferred by SysAdmins when managing a consistent Linux environment as the exact executable used is more predictable
* But if the script needs to run on other distributions, e.g. Solaris, where the location of bash might not necessary be the same, `#!/usr/bin/env bash` should be used.

## Always Quote
### Motivation
Given:
```bash
$ ls -l
total 0
-rw-r--r--  1 user  user  0 Oct 30 22:24 file
-rw-r--r--  1 user  user  0 Oct 30 22:24 file backup
```
What is the result of the following commands:
```bash
$ backup='file backup'
$ rm ${backup}
$ ls -l
```
```
a) Only 'file' is present
b) Only 'file backup' is present
c) Both files are present
d) Both files are absent
```

### Wordsplitting
Upon receiving your command, the shell performs wordsplitting on the arguments before executing the command.

By default, the arguments will be split into words, using consecutive whitespace as the delimiter.

Thus, in the following, 4 arguments `one`, `two`, `threeA` and `threeB` will be passed into command:
```bash
$ var='threeA threeB'
$ command one two ${var}
```

Quoting is the way to prevent wordsplitting from occurring, only 3 arguments `one`, `two` and `threeA threeB` will be passed into command:
```bash
$ var='threeA threeB'
$ command one two "${var}"
```

### Pathname Expansion/ Globbing
In addition to wordsplitting, the shell performs pathname expansion (globbing) after wordsplitting.

The following command will receive an unexpected number of arguments depending on how many files are in the current directory:
```bash
$ var='* whee'
$ command one two ${var}
```
Replace command with `rm` and it's not hard to see why unquoted variables are said to be ticking timebombs.

### Single Quotes vs Double Quotes
* Single Quoting is the strictest, only single quotes pose a problem within single quotes
* Double Quoting allows variable expansion (and some other types of expansion), but inhibits wordsplitting and pathname expansion
* General rule of thumb would be to always use single quotes and fallback to double quotes when we need variable expansion.

### Prefer Brace-Quoting for String Interpolation
* Main reason to brace quote is to prevent ambiguity during variable expansion:
    ```bash
    $ echo "${file}suffix"
    ```
* For positional arguments and special variables, possible convention is to not use brace: "$1", "$2", "$@", "$#", "$?", "$$", etc.

### Prefer "$()" over `` for Command Substitution
* Allows us to use the output of a command for another command:
    ```bash
    $ echo "Output of command: $(command "${var}")"
    ```
* Main advantage of "$()" over `` is ease of quoting


## Use Arrays
### Motivation
Given:
```bash
$ dynamic_args='--author=user'
$ dynamic_args="${dynamic_args}"' --title=bash scripting'
$ command ${dynamic_args}
```
How many arguments are passed to command?
```
a) 1
b) 2
c) 3
d) 4
```
What about this:
```bash
$ dynamic_args='--author=user'
$ dynamic_args="${dynamic_args}"' --title=bash scripting'
$ command "${dynamic_args}"
```

### Arrays Allow Handling of Multiple Arguments with Proper Quoting 
* Always use quoted form "${array[@]}"
* This form will expand each element in the array into a properly quoted argument:
    ```bash
    $ dynamic_args=('--author=user')
    $ dynamic_args+=('--title=bash scripting')
    $ command "${dynamic_args[@]}"
    ```


## Conditionals
### Motivation
Given that we are interested in doing some alphabetic comparison:
```bash
$ var='a'
$ [ "${var}" > 'z' ] && echo 'a comes after z'
```
What is the output of this command?
```
a) 'a comes after z'
b) no output
c) -bash: z: No such file or directory
d) none of the above
```

### Prefer [[ over [ or test
* `[[` is shell syntax as compared to `[` or `test` which are shell commands
* Besides avoiding the issues in the preceding example, it also allows for pattern matching:
    ```bash
    $ var='somethingwithsuffix'
    $ [[ "${var}" == *suffix ]] && echo 'has suffix'
    ```
* And regex matching
    ```bash
    $ var='somethingwithsuffix'
    $ [[ "${var}" =~ .*suffix$ ]] && echo 'has suffix'
    ```

### Arithmetic Evaluation
* By default, logical operators like `<`, `>=` within `[[` work on string values, to compare numbers, we have to use arguments like `-lt`, `-ge`.
* Bash has a more familiar syntax for arithmetic evaluation:
    ```bash
    $ (( 1 > 2 )) && echo '1 is greater than 2'
    ```


## Handle Exit Status
### Scripts do not Exit on Error
By default, scripts do not exit on error like we are used to in programming languages:
```bash
#!/bin/bash

cd nonExistingDirectory
rm *
```
The cd will fail and the files of the current directory will be wrongly removed.

Handle the errors by chaining `&&` or by exiting explicitly on error:
```bash
#!/bin/bash

cd nonExistingDirectory && rm *
cd anotherNonExistingDirectory || exit "$?"
rm *
```

### What about `set -o errexit` or Unofficial Bash Strict Mode?
* There are some pitfalls of errexit that make its behaviour inconsistent
* This makes it kind of like working in a programming language that "sometimes" throw exceptions
* Thus, the preference to handle errors manually
* However, if one would like to use it, it would be important to first familiarise with its [pitfalls](https://mywiki.wooledge.org/BashFAQ/105)
* The same goes for the [Unofficial Bash Strict Mode](http://redsymbol.net/articles/unofficial-bash-strict-mode/)

## I/O & Redirection
### Motivation
Given the following `echo-std` script:
```bash
#!/bin/bash

printf 'stdout appears! '
printf 'stderr appears!\n' >&2
```
What is the output of the following command:
```bash
$ ./echo-std 2>&1 >/dev/null
```
```
a) no output
b) stdout appears!
c) stderr appears!
d) stdout appears! stderr appears!
```

### Understanding Redirection
* `>` is equivalent to `1>`
* So `>file` will redirect FD1 which is stdout to `file`
* `2>&1` is best understood as the duplicating FD syntax where the value in FD1 is copied into FD2
* See [Order Of Redirection, i.e., "> file 2>&1" vs. "2>&1 >file"](https://wiki.bash-hackers.org/howto/redirection_tutorial#order_of_redirection_ie_file_2_1_vs_2_1_file) for illustration


## Tools
* [shellcheck](https://github.com/koalaman/shellcheck)
    - Static analysis tool for shell scripts
    - Plugins available in IntelliJ, vim, VSCode, etc.
* [bats-core](https://github.com/bats-core/bats-core)
    - Automated testing tool for shell scripts
    - Assertion Library: [bats-assert](https://github.com/jasonkarns/bats-assert-1)


## References
* [Greg's BashFAQ](https://mywiki.wooledge.org/BashFAQ)
* [Greg's Bash Pitfalls](https://mywiki.wooledge.org/BashPitfalls)
* [Greg's BashGuide](https://mywiki.wooledge.org/BashGuide)
* [Safe ways to do things in bash](https://github.com/anordal/shellharden/blob/master/how_to_do_things_safely_in_bash.md)
* [Google Shell Style Guide](https://google.github.io/styleguide/shell.xml)
