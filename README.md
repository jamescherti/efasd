# eFasd (Enhanced Fasd) - A fasd fork that is optimized for faster shell prompt speed

- Maintainer: James Cherti <[https://www.jamescherti.com/](https://www.jamescherti.com/)>
- License: MIT

Navigating deeply nested file systems in POSIX shells on Linux and UNIX systems often requires typing repetitive, lengthy paths.

The **[eFasd](https://github.com/jamescherti/efasd)** project resolves this by maintaining a continuous record of the files and directories you access.

Instead of writing out absolute paths or executing multiple `cd` commands, you can use default `fasd` aliases like `z` to jump to a directory, `f` for files, `d` for directories, and `s` for search. This lets you navigate directly to active project directories or open common configuration files quickly.

The `fasd` command calculates a "frecency" score for each path using an algorithm that weights both how often and how recently a file or directory was opened. This allows it to predict your target destination with minimal input.

This utility is inspired by tools like autojump, z, and v. The name `fasd` comes from the default aliases: `f` (files), `a` (files/directories), `s` (show/search/select), and `d` (directories).

## About this fork (eFasd)

The eFasd project is a fork of the *whjvenyl/fasd* repository. Because the original repository is unmaintained, this fork incorporates performance and reliability fixes:

- For Bash environments, eFasd eliminates prompt latency by wrapping the execution pipeline in a double-fork pattern within `bash-hook`. This pushes the `fasd` database processing to the background, preventing prompt blocking while cleanly suppressing Bash job control output notifications. Upstream *whjvenyl/fasd* executes synchronously during prompt evaluation, causing approximately 35ms of latency per prompt render from subshell forks and disk I/O.
- Implemented strict signal traps to guarantee temporary files are cleaned up on unexpected signal interruptions or process termination. This prevents orphaned temporary files from accumulating in the filesystem.
- Fixed a bug in the `get_vcs` directory traversal logic that triggered an infinite loop when encountering unreadable or restricted parent directories.
- Added automatic filtering of temporary directories from the `fasd` database, preventing temporary paths from polluting search rankings.

## Introduction

If you use your shell to navigate and launch applications, fasd enables you to open files and directories regardless of your current working directory:

```
v def conf       =>     vim /some/awkward/path/to/type/default.conf
z abc            =>     cd /hell/of/an/awkward/path/to/get/to/abcdef
m movie          =>     mpv /whatever/whatever/whatever/awesome_movie.mp4
o eng paper      =>     xdg-open /you/dont/remember/where/english_paper.pdf
vim `f rc lo`    =>     vim /etc/rc.local
vim `f rc conf`  =>     vim /etc/rc.conf
```

Fasd defines the following aliases by default:

```sh
alias a='fasd -a'        # any
alias s='fasd -si'       # show / search / select
alias d='fasd -d'        # directory
alias f='fasd -f'        # file
alias sd='fasd -sid'     # interactive directory selection
alias sf='fasd -sif'     # interactive file selection
alias z='fasd_cd -d'     # cd into directory
alias zz='fasd_cd -d -i' # cd with interactive selection
```

Fasd detects whether to display a list of matches or only the top result. When called inside a subshell with query arguments, it outputs only the highest-ranked match:

```sh
mv update.html `d www`
cp `f mov` .
```

## Comparison with Zoxide

Both eFasd and Zoxide rank paths by frecency (frequency and recency), allowing you to access deep filesystem paths using short fuzzy queries instead of typing out full paths.

While Zoxide is a compiled Rust binary focused solely on directory navigation, eFasd tracks both files and directories. Because eFasd is written entirely in POSIX sh with zero binary dependencies, it runs anywhere a standard shell exists. You can drop it onto stripped-down containers, legacy enterprise servers, or embedded devices without worrying about CPU architectures, libc versions, or approval processes for third-party binaries.

## Install

Download the script directly:

```sh
curl -L -o fasd https://raw.githubusercontent.com/jamescherti/efasd/master/fasd
chmod +x fasd
```

Or clone the repository:

```sh
git clone https://github.com/jamescherti/efasd
```

A Makefile is included to install `fasd` to system paths.

System-wide install:

```sh
make install
```

Install to `$HOME`:

```sh
PREFIX=$HOME make install
```

Alternatively, copy `fasd` to any directory in your `$PATH`.

### Shell Initialization

To initialize fasd, add the following command to your shell configuration file (`~/.bashrc`, `~/.zshrc`, etc.):

```sh
eval "$(fasd --init auto)"
```

This registers a command hook that tracks accessed paths and configures tab completion for supported shells.

If you prefer explicit control over your environment, pass custom flags to `fasd --init`:

```
zsh-hook             # define _fasd_preexec and add it to zsh preexec array
zsh-ccomp            # zsh command mode completion definitions
zsh-ccomp-install    # setup command mode completion for zsh
zsh-wcomp            # zsh word mode completion definitions
zsh-wcomp-install    # setup word mode completion for zsh
bash-hook            # add backgrounded hook code to bash $PROMPT_COMMAND
bash-ccomp           # bash command mode completion definitions
bash-ccomp-install   # setup command mode completion for bash
posix-alias          # define aliases that apply to all posix shells
posix-hook           # setup $PS1 hook for posix-compatible shells
tcsh-alias           # define aliases for tcsh
tcsh-hook            # setup tcsh precmd alias
```

Example of a minimal zsh setup without tab completion:

```sh
eval "$(fasd --init posix-alias zsh-hook)"
```

To eliminate shell startup latency caused by evaluating the initialization script on launch, cache the init output in your `.bashrc`:

```sh
fasd_cache="$HOME/.fasd-init-bash"
if [ "$(command -v fasd)" -nt "$fasd_cache" -o ! -s "$fasd_cache" ]; then
  fasd --init posix-alias bash-hook bash-ccomp bash-ccomp-install >| "$fasd_cache"
fi
source "$fasd_cache"
unset fasd_cache
```

You can also source `fasd` directly if you want it to operate as a shell function rather than an external binary.

To use custom directory jumping aliases (such as `j` instead of `z`):

```sh
alias j='fasd_cd -d'
```

## Examples

```sh
f foo           # list frecent files matching foo
a foo bar       # list frecent files and directories matching foo and bar
f js$           # list frecent files that end in js
f -e vim foo    # run vim on the most frecent file matching foo
mpv `f bar`     # run mpv on the most frecent file matching bar
z foo           # cd into the most frecent directory matching foo
open `sf pdf`   # interactively select a file matching pdf and launch open
```

Custom aliases can be defined to bind frecent lookups to specific programs:

```sh
alias v='f -e vim'      # open matching file with vim
alias m='f -e mpv'      # open matching media with mpv
alias o='a -e xdg-open' # open matching target with xdg-open
```

In Bash, register completion for custom aliases using `_fasd_bash_hook_cmd_complete`:

```sh
_fasd_bash_hook_cmd_complete v m z o
```

## Matching

Fasd provides three matching modes: default, case-insensitive, and fuzzy.

For a given set of command-line queries, a path is considered a match if:

1. Queries match segments of the path in sequential order.
2. The final query matches the last segment of the path.

If no match is found, fasd repeats the evaluation with case sensitivity disabled. If no result matches, it introduces character skipping based on the fuzzy threshold.

Tips:

- To match a path where the final query is not the last segment, append `/` as the final query argument.
- To match against the exact end of a filename, append `$` to the final query argument.

## How It Works

When initialized, fasd hooks into the shell prompt cycle. The hook parses the arguments of executed commands and verifies whether they correspond to existing files or directories. Matching paths are recorded and updated in the frecency index.

## Compatibility

Fasd core functionality is POSIX compliant. Shell tracking relies on command substitution within `$PS1` or shell-specific hooks (`preexec` / `PROMPT_COMMAND`).

Fasd has been tested across:

- bash
- zsh
- mksh
- pdksh
- dash
- busybox ash
- FreeBSD /bin/sh
- OpenBSD /bin/sh

## Synopsis

```
fasd [options] [query ...]
[f|a|s|d|z] [options] [query ...]
  options:
    -s         list paths with scores
    -l         list paths without scores
    -i         interactive mode
    -e <cmd>   set command to execute on the result file
    -b <name>  only use <name> backend
    -B <name>  add additional backend <name>
    -a         match files and directories
    -d         match directories only
    -f         match files only
    -r         match by rank only
    -t         match by recent access only
    -R         reverse listing order
    -h         show a brief help message
    -[0-9]     select the nth entry

fasd [-A|-D] [paths ...]
    -A         add paths
    -D         delete paths
```

## Tab Completion

Fasd supports command mode completion in Bash and Zsh, and word mode completion in Zsh.

Command mode completion triggers on standard Tab key presses after typing `fasd` or any of its configured aliases. Under this mode your queries can be separated by a space. Tip: if you find that the completion result overwrites your queries, type an extra space before you hit tab.

Word mode completion in Zsh triggers anywhere on the command line using prefix or suffix markers:

- Prefix triggers: `,` (all), `f,` (files), `d,` (directories)
- Suffix triggers: `,,` (all), `,,f` (files), `,,d` (directories)

Examples:

```sh
$ vim ,rc,lo<Tab>
$ vim /etc/rc.local

$ mv index.html d,www<Tab>
$ mv index.html /var/www/

There are also three zle widgets: `fasd-complete`, `fasd-complete-f`, `fasd-complete-d`. You can bind them to keybindings you like:

```sh
bindkey '^X^A' fasd-complete    # C-x C-a to do fasd-complete (files and directories)
bindkey '^X^F' fasd-complete-f  # C-x C-f to do fasd-complete-f (only files)
bindkey '^X^D' fasd-complete-d  # C-x C-d to do fasd-complete-d (only directories)
```

## Backends

Fasd can ingest recent path history from external sources:

- `spotlight`: macOS Spotlight index (entries accessed or modified recently).
- `recently-used`: GTK recently-used.xbel file history.
- `current`: Entries located within the current `$PWD`.
- `viminfo`: History from `$HOME/.viminfo`.

You can set custom backends using `_FASD_BACKENDS` in your configuration file.

You can define your own backend by declaring a function by that name in your [config file](#tweaks). You can set default backend with `_FASD_BACKENDS` variable in our [config file](#tweaks).

## Tweaks

Configuration settings can be defined in `$HOME/.config/fasd/config` (or `$XDG_CONFIG_HOME/fasd/config`):

```
$_FASD_DATA
Path to the fasd data file, default "$XDG_CACHE_HOME/fasd" or
"$HOME/.cache/fasd" if $XDG_CACHE_HOME is not set.

$_FASD_BLACKLIST
List of blacklisted strings. Commands matching them will not be processed.
Default is "--help".

$_FASD_SHIFT
List of all commands that needs to be shifted, defaults to "sudo busybox".

$_FASD_IGNORE
List of all commands that will be ignored, defaults to "fasd ls echo".

$_FASD_TRACK_PWD
Fasd defaults to track your "$PWD". Set this to 0 to disable this behavior.

$_FASD_ONLY_VCS
Resolves paths within Git or Mercurial repositories to the repository root. Defaults to 1.

$_FASD_AWK
Which awk to use. Fasd can detect and use a compatible awk.

$_FASD_SINK
File to log all STDERR to, defaults to "/dev/null".

$_FASD_MAX
Max total score / weight, defaults to 2000.

$_FASD_SHELL
Which shell to execute. Some shells will run faster than others. fasd
runs faster with dash and ksh variants.

$_FASD_BACKENDS
Default backends.

$_FASD_RO
If set to any non-empty string, fasd will not add or delete entries from
database. You can set and export this variable from command line.

$_FASD_FUZZY
Level of "fuzziness" when doing fuzzy matching. More precisely, the number of
characters that can be skipped to generate a match. Set to empty or 0 to
disable fuzzy matching. Default value is 2.

$_FASD_VIMINFO
Path to .viminfo file for viminfo backend, defaults to "$HOME/.viminfo"

$_FASD_RESOLVE_SYMLINKS
Set this option to 1 if you want fasd to resolve symbolic links. This is useful
to prevent duplicates in the entry list. Defaults to 0.

$_FASD_IGNORE_TMP
Set to 0 to disable the filtering of temporary directories. Defaults to 1.

$_FASD_RECENTLY_USED_XBEL
Path to XDG recently-used.xbel file for recently-used backend, defaults to
"$HOME/.local/share/recently-used.xbel"

$_FASD_NOCASE
If set to any non-empty string, fasd will ignore case when matching.
```

## Debugging

If fasd does not work as expected, please file a bug report describing the unexpected behavior along with your OS version, shell version, awk version, sed version, and a log file.

You can set `_FASD_SINK` in your [config file](#tweaks) to obtain a log.

```sh
_FASD_SINK="$HOME/.fasd.log"
```

## Articles

- **[Speed-up fasd shell prompt on UNIX and Linux systems](https://www.jamescherti.com/speed-up-fasd-shell-prompt/)**: Depending on the fasd implementation and shell hook, tracking can delay prompt rendering. This article describes environment variables that can reduce fasd shell prompt overhead: Version Control System (VCS) directory traversal, database size limits, symlink resolution, and shell string parsing dependencies.

- **[Prioritizing fasd directories in bash, sh, and zsh shells (e.g., pinned and project directories)](https://www.jamescherti.com/fasd-prioritize-pinned-directories-bash-zsh-shell/)**: While the default fasd tracking works well for day-to-day shell navigation, there are a few edge cases when you have a set of core directories you always want at the top of your results. For example, if you clone a new repository, provision a new machine, or generate a directory list automatically, fasd knows nothing about those paths until you manually cd into each one first. To solve this, you can use a custom fasd backend to "pin" a static list of directories so they permanently outrank regular shell history.

## Links

- [efasd @GitHub](https://github.com/jamescherti/efasd)
- [quick-fasd.el](https://github.com/jamescherti/quick-fasd.el): An Emacs package to integrate `fasd`.
