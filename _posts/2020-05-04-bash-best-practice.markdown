---
layout: post
title:  "Bash Best Practice - Part 1"
author: "Andreas Heumaier"
date:   2020-05-04 11:01:07 +0200
categories: tech
---

Since Bash has several [known weaknesses](https://mywiki.wooledge.org/BashWeaknesses) I don't enjoy writing such code that much. It's often used when all you want is automate a mundane task. *"I'll just copy/paste the commands I usually run, add a few IFs and FORs and that will be enough"*.  Well, that's how many shell scripts come to existence I guess, but nonetheless writing such scripts is not as easy as it sounds, and there are many pitfalls to avoid. 

The shell is a paradox. It is mysterious in its complexity and marvelous in its simplicity. You can learn the bare basics with just of couple hours of tinkering, but becoming a shell master takes years of consistent use, and even then it feels like you learn something new every day. On one hand, it has an ugly syntax, and trying to work with strings and arrays is beyond frustrating. On the other hand, there’s nothing quite like the zen achieved when you stitch together a half dozen shell commands to extract and format your data of interest, creating a pipeline that outperforms any Python or C++ code you could ever write.

Whatever your feelings about the shell in general, or about (the most popular shell) Bash in particular, you cannot deny that it is a rustic language with very poor debugging facilities that makes it very easy to shoot yourself in the foot. If you can avoid writing code in Bash, you should. In fact, as some say, if you can avoid writing code at all, you should. But if you’re a modern engineer then chances are that sooner or later the shell will be the best tool for a particular job. What do you do then?

The best Bash scripts not only work, but are written in such a way that they are easy to understand and modify. A lot of this comes from [using consistent names for variables and a consistent coding style](https://github.com/azet/community_Bash_style_guide). The principles of [Clean Code](https://www.pearson.com/us/higher-education/program/Martin-Clean-Code-A-Handbook-of-Agile-Software-Craftsmanship/PGM63937.html) apply to Bash as well. 

So when to use Bash and when to avoid Bash?
It's rather simple:

- Does it need to glue userland utilities together? Use Bash.
- Does it need to do complex tasks (e.g. database queries, matrix like data structures)? 
Use something else.

Why? You can do a lot of complicated tasks with Bash, and I've had some experience in trying them all out in Bash. It consumes a lot of time and is often very difficult to debug in comparison to dynamic programming languages such as python. You are simply going to waste valuable time, performance and nerve you could have spent better otherwise.

Over the last few months I’ve really tried to look for ways to improve my shell code. So the next time you absolutely, positively have to write some shell code, consider referring to these resources to improve the experience!

## Coding style

Sometimes you'll come across a .sh script with a `/bin/sh` shebang. (That is, a file that starts with `#!/bin/sh`). I believe you should not do that, unless you know what you are doing. If your script starts with `#!/bin/sh`, it's telling the operating system that the script should be run with the `/bin/sh` binary. POSIX says that `/bin/sh` should exist and point to a “POSIX compliant” shell. But on debian, it's a symlink to /bin/dash, and on Arch Linux, it's a symlink to `/usr/bin/Bash`.
So if you use a `#!/bin/sh` shebang, be prepared to get weird errors when switching distributions, or prove yourself that the code you wrote is indeed “POSIX”. I find it much easier to just stick a `#/bin/Bash` shebang and call it a day.  
Hint: It seems Bash will still do The Right Thing (tm) if it detects that `argv[0]` is `/bin/sh`

### Use Bash Strict Mode (Unless You Love Debugging)

Bash has a lot of “switches” you can activate with the set built-in. (Type set -o to get a list of them).
Abort the script on errors and undbound variables. Put the following code at the beginning of each script.

{% highlight bash %}
#!/bin/bash
# Exit on error. Append "|| true" if you expect an error.
set -o errexit
# Exit on error inside any functions or subshells.
set -o errtrace
# Do not allow use of undefined vars. Use ${VAR:-} to use an undefined VAR
set -o nounset
# Catch the error in case mysqldump fails (but gzip succeeds) in `mysqldump |gzip`
set -o pipefail
# Turn on traces, useful while debugging but commented out by default
# set -o xtrace

# Internal Field Separator - controls what Bash calls word splitting.
IFS=$'\n\t'  
{% endhighlight %}

A shorter version is shown below, but writing it out makes the script more readable.

{% highlight bash %}
#!/bin/bash
set -euo pipefail
IFS=$'\n\t'
{% endhighlight %}

I call this the unofficial bash strict mode. This causes bash to behave in a way that makes many classes of subtle bugs impossible. You'll spend much less time debugging, and also avoid having unexpected complications in production.

There is a short-term downside: these settings make certain common bash idioms harder to work with. Most have simple workarounds, described in more detailed here: [Issues & Solutions](http://redsymbol.net/articles/unofficial-bash-strict-mode/#issues-and-solutions).

Hint: if you do want to allow a command to fail, you can simply use a `|| true` to do the trick:

{% highlight bash %}
set -o errexit
cd path/to/foo
command-that-may-fail || true
{% endhighlight %}

{% highlight bash %}
$ Bash foo.sh
foo.sh: line 4: my_optoin: unbound variable
{% endhighlight %}

Hint: you should really use `printf '%s\n' "$my_option"` instead to avoid problems if for instance my_option is -e

## Variables

### Style

- Prefer local variables within functions over global variables
- If you need global variables, make them readonly like:  `declare -r MYVAR='value'`
- Variables should always be quoted, especially if their value may contain a whitespace or separator character: `"${var}"`
- Capitalization:
    - Environment (exported) variables: `${ALL_CAPS}`
    - Local variables: `${lower_case}`
- Positional parameters of the script should be checked, those of functions should not
- Some loops happen in subprocesses, so don’t be surprised when setting variabless does nothing after them. Use stdout and `grep`-ing to communicate status.
- Use a single equal sign when checking `if [[ "${NAME}" = "Kevin" ]];` double or triple signs are not needed.
- Use the new bash builtin test operator `([[ ... ]])` rather than the old single square bracket test operator or explicit call to test.

### Parameter notation 

Always use long parameter notation when available. This makes the script more readable, especially for lesser known/used commands that you don't remember all the options for.
If you are on the CLI, abbreviations make sense for efficiency. Nevertheless, when you are writing reusable scripts, a few extra keystrokes will pay off in readability and avoid ventures into man pages in the future, either by you or your collaborators. Similarly, we prefer `set -o nounset` over `set -u`.

{% highlight bash %}
# Avoid:
rm -rf -- "${dir}"

# Good:
rm --recursive --force -- "${dir}"
{% endhighlight %}

### Safety and Portability

1. Use `{}` to enclose your variables. Otherwise, Bash will try to access the `$ENVIRONMENT_app` variable in `/srv/$ENVIRONMENT_app`, whereas you probably intended `/srv/${ENVIRONMENT}_app`. Since it is easy to miss cases like this, we recommend that you make enclosing a habit.
2. Use set, rather than relying on a shebang like `#!/usr/bin/env bash -e`, since that is neutralized when someone runs your script as `bash yourscript.sh`.
3. Use #!/usr/bin/env bash, as it is more portable than #!/bin/bash.
Use `${BASH_SOURCE[0]}` if you refer to current file, even if it is sourced by a parent script. In other cases, use `${0}`.
4. Use `:-` if you want to test variables that could be undeclared. For instance, with if `[[ "${NAME:-}" = "Kevin" ]]`, `$NAME` will evaluate to *Kevin* if the variable is empty. The variable itself will remain unchanged. The syntax to assign a default value is `${NAME:=Kevin}`.



## Substitution

- Always use `$(cmd)` for command substitution (as opposed to backquotes)
- Prepend a command with `\` to override alias/builtin lookup. E.g.:

    {% highlight bash %}
    $ \time Bash -c "dnf list installed | wc -l"
    5466
    1.32user 0.12system 0:01.45elapsed 99%CPU (0avgtext+0avgdata 97596maxresident)k
    0inputs+136outputs (0major+37743minor)pagefaults 0swaps
    {% endhighlight %}

## Output and redirection

- [For various reasons](https://www.in-ulm.de/~mascheck/various/echo+printf/), `printf` is preferable to `echo`. `printf` gives more control over the output, it's more portable and its behaviour is defined better.
- Print error messages on stderr. E.g., I use the following function:

    {% highlight bash %}
    error() {
      printf "${red}!!! %s${reset}\\n" "${*}" 1>&2
    }
    {% endhighlight %}

- Name heredoc tags with what they’re part of, like:

    {% highlight bash %}
    cat <<HELPMSG
    usage $0 [OPTION]... [ARGUMENT]...

    HELPMSG
    {% endhighlight %}

- Single-quote heredocs leading tag to prevent interpolation of text between them.

    {% highlight bash %}
    cat <<'MSG'
    [...]
    MSG
    {% endhighlight %}

- When combining a `sudo` command with redirection, it's important to realize that the root permissions only apply to the command, not to the part after the redirection operator. An example where a script needs to write to a file that's only writeable as root:

    {% highlight bash %}
    # this won't work:
    sudo printf "..." > /root/some_file

    # this will:
    printf "..." | sudo tee /root/some_file > /dev/null
    {% endhighlight %}

### Avoid useless pipes

Very often you can get rid of a pipe if you use the correct syntax. Here are some examples:
{% highlight bash %}
# you want to replace 'foo' by 'bar' in the
# value of $my_var:

# bad
my_new_var=$(echo $my_var | sed -e s/foo/bar/)
# better
my_nev_var=${my_var/foo/bar}
{% endhighlight %}

The last example is one of the many things you can do with Bash variables. Here's a [list of the parameter substitutions](https://www.tldp.org/LDP/abs/html/parameter-substitution.html)] you can use.

## Functions

Bash can be hard to read and interpret. Using functions strongly improve readability. Principles from [Clean Code](https://www.pearson.com/us/higher-education/program/Martin-Clean-Code-A-Handbook-of-Agile-Software-Craftsmanship/PGM63937.html) apply here too.

- Apply the [Single Responsibility Principle](https://en.wikipedia.org/wiki/Single_responsibility_principle): a function does one thing.
- [Don't mix levels of abstraction](https://sivalabs.in/2013/12/clean-code-dont-mix-different-levels-of-abstractions/)
- Describe the usage of each function: number of arguments, return value, output
- Declare variables with a meaningful name for positional parameters of functions

    {% highlight bash %}
    foo() {
      local first_arg="${1}"
      local second_arg="${2}"
      [...]
    }
    {% endhighlight %}

- Create functions with a meaningful name for complex tests

    {% highlight bash %}
    # Don't do this
    if [ "$#" -ge "1" ] && [ "$1" = '-h' ] || [ "$1" = '--help' ] || [ "$1" = "-?" ]; then
      usage
      exit 0
    fi

    # Do this
    help_wanted() {
      [ "$#" -ge "1" ] && [ "$1" = '-h' ] || [ "$1" = '--help' ] || [ "$1" = "-?" ]
    }

    if help_wanted "$@"; then
      usage
      exit 0
    fi
    {% endhighlight %}

### Function packaging
It is nice to have a Bash package that can not only be used in the terminal, but also invoked as a command line function. In order to achieve this, the exporting of your functionality should follow this pattern:

{% highlight bash %}
if [[ "${BASH_SOURCE[0]}" = "${0}" ]]; then
  my_script "${@}"
  exit $?
fi
export -f my_script
{% endhighlight %}

This allows a user to source your script or invoke it as a script.

{% highlight bash %}
# Running as a script
$ ./my_script.sh some args --blah

# Sourcing the script
$ source my_script.sh
$ my_script some more args --blah
{% endhighlight %}

### Use sub-shells

Let's say you want to run the make command in all the subdirectories of your current working directory.
{% highlight bash %}
proj_1
|_ Makefile
|_ proj_1.c
proj_2
|_ Makefile
|_ proj_1.c
{% endhighlight %}

You may start by writing:
{% highlight bash %}
for project in */; do
  cd ${project} && make
done
{% endhighlight %}
But that won't work. After cd proj_1, you must go back to the top directory so that cd proj_2 can work. You could workaround that using popd and pushd that allow you to maintain a “stack” of working directories

{% highlight bash %}
pushd "${foo}"
[...]
popd
{% endhighlight %}

but there's an easier way:

{% highlight bash %}
for project in */; do
  (cd ${project} && make)
done
{% endhighlight %}
By using parentheses, you've created a “sub-shell” that won't interfere with the main script.

## Cleanup on exit

An idiom for tasks that need to be done before the script ends (e.g. removing temporary files, etc.). The exit status of the script is the status of the last statement *before* the `finish` function.

{% highlight bash %}
scratch=$(mktemp -d -t tmp.XXXXXXXXXX)

finish() {
  result=$?
  # Your cleanup code here e.g.
  rm -rf "$scratch"
  exit ${result}
}

trap finish EXIT ERR
# Now your script can write files in the directory "$scratch".
# It will automatically be deleted on exit, whether that's due
# to an error, or normal completion.
{% endhighlight %}

Source: Aaron Maxwell, [How "Exit Traps" can make your Bash scripts way more robust and reliable](http://redsymbol.net/articles/Bash-exit-traps/).



## Use static analysis

Yes, you can do this for Bash scripts too :). I like to use [shellcheck](https://github.com/koalaman/shellcheck) for this. Here's a sample of what shellcheck can do:

{% highlight bash %}

In foo.sh line 40:
find . -name "*.back" | xargs rm
^-- SC2038: Use -print0/-0 or -exec + to allow for non-alphanumeric filenames.

read name
^-- SC2162: read without -r will mangle backslashes.

$bin/foo bar.txt
^-- SC2086: Double quote to prevent globbing and word splitting.

my_cmd *
^-- SC2035: Use ./*glob* or -- *glob* so names with dashes won't become 
options.
{% endhighlight %}

The best thing about shellcheck is that each error message leads you to a detailed page explaining the issue.

Another one is [shellfmt](https://github.com/mvdan/sh)  - you guess it it is stolen from gofmt

## Be careful with coreutils

The so-called coreutils (cp, mv, ls, …) come with various flavours. Basically, there's the “GNU” and the “BSD” flavors, so be careful to not use things that only work in the “GNU” version.

This can happen when you switch from linux to OSX or vice-versa.

(for instance cp foo.txt bar.txt --verbose will not work on OSX, you have to put the option --verbose before the arguments)



## Shell script template

An annotated template for Bash shell scripts:

For now, see <https://github.com/bertvv/dotfiles/blob/master/.vim/templates/sh>


### Templates

- Bash-script-template <https://github.com/ralish/Bash-script-template>
- Bash3 Boilerplate <http://Bash3boilerplate.sh/>

### Portable shell scripts

- <https://wiki.ubuntu.com/DashAsBinSh>
- <http://pubs.opengroup.org/onlinepubs/009695399/utilities/contents.html>
- <http://sites.harvard.edu/~lib113/reference/unix/portable_scripting.html>
- <https://www.gnu.org/software/autoconf/manual/autoconf.html#Portable-Shell>


## Resources

- Araps, Dylan (2018). *Pure Bash Bible.* <https://github.com/dylanaraps/pure-Bash-bible>
- Armstrong, Paul (s.d.). *Shell Style Guide.* <https://google.github.io/styleguide/shell.xml>
- Bash Hackers Wiki. <http://wiki.Bash-hackers.org/start>
- Bentz, Yoann (2016). *Good practices for writing shell scripts.* <http://www.yoone.eu/articles/2-good-practices-for-writing-shell-scripts.html>
- Berkholz, Donny (2011). *Bash shell-scripting libraries.* <https://dberkholz.com/2011/04/07/Bash-shell-scripting-libraries/>
- Billemont, Maarten (2017). The Bash Guide. <http://guide.Bash.academy/>
- Brady, Pádraig (2008). *Common Shell Script Mistakes.* <http://www.pixelbeat.org/programming/shell_script_mistakes.html>
- Cooper, Mendel (2014). *The Advanced Bash Scripting Guide (ABS).* <http://www.tldp.org/LDP/abs/html/>
- Fox, Brian and Ramey, Chet (2009). *Bash(1) man page.* <http://linux.die.net/man/1/Bash>
- Free Software Foundation (2014). *Bash Reference Manual.* <https://www.gnu.org/software/Bash/manual/Bashref.html>
- Gite, Vivek (2010). *Linux Shell Scripting Tutorial (LSST) v2.0.* <https://Bash.cyberciti.biz/guide/>
- GreyCat (Ed.) (2015). *Bash Guide.* <http://mywiki.wooledge.org/BashGuide>
- GreyCat (Ed.) (2017). *Bash Frequently Asked Questions.* <https://mywiki.wooledge.org/BashFAQ>
- GreyCat (Ed.) (2020). *Bash Pitfalls.* <https://mywiki.wooledge.org/BashPitfalls>
- Jones, M. Tim (2011). *Evolution of shells in Linux: From Bourne to Bash and beyond.* <https://www.ibm.com/developerworks/library/l-linux-shells/>
- Lavi, Kfir (2012). *Defensive Bash Programming.* <http://www.kfirlavi.com/blog/2012/11/14/defensive-Bash-programming>
- Maxwell, Aaron (2014). *Use the Unofficial Bash Strict Mode (Unless You Looove Debugging)*. <http://redsymbol.net/articles/unofficial-Bash-strict-mode/>
- Pennarun, Avery (2011). *Insufficiently known POSIX shell features.* <http://apenwarr.ca/log/?m=201102#28>
- Rousseau, Thibaut (2017). **Shell Scripts Matter.** <https://dev.to/thiht/shell-scripts-matter>
- Sheppard, Simon (s.d.). *Bash Keyboard Shortcuts.* <http://ss64.com/Bash/syntax-keyboard.html>
- When to use Bash: <https://hackaday.com/2017/07/21/linux-fu-better-Bash-scripting/#comment-3793634>



Learn in the [next chapter](https://www.minimalmistakes.de/bash/2020/05/11/bash-debuggin-best-practice.html) how to debup shell scripts properl.y 