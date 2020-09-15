---
title: Useful Shell Commands
published: true
description: If you're new to development, you'll want to keep these shell commands in your toolbelt for all sorts of handy scripts
tags: [bash,productivity,beginners]
---

## 1) ls

Need to figure out what is in a directory? `ls` is your friend. It will list out the contents of a directory and has a number of flags to help control how those items are displayed. Since the default `ls` doesn't display entries that begin with a `.`, you can use `ls -a` to make sure to include those entries as well.

```
nyxtom@enceladus$ ls -a
./                              README.md                       _dir_colors                     _tern-project                   _vimrc                          install.sh*
../                             _alacritty-theme/               _gitconfig                      _tmux/                          alacritty-colorscheme*
.git/                           _alacritty.yml                  _profile                        _tmux.conf
.gitignore                      _bashrc                         _terminal/                      _vim/                           imgcat.sh*
```

Need it in a 1 column layout (one entry per line)? Use `ls -1`. Need to include a longer format with size, permissions, and timestamps use `ls -l`. Need those entries sorted by last changed use `ls -l -t`. Need to recursively list them? Use `ls -R`. Want to sort by file size? Use `ls -S`.

## 2) cat

Need to output the contents of a file. Use `cat`! Bonus: use `cat -n` to include numbers on the lines.

```
nyxtom@enceladus$ cat -n -s _dir_colors
     1  .red 00;31
     2  .green 00;32
     3  .yellow 00;33
     4  .blue 00;34
     5  .magenta 00;35
     6  .cyan 00;36
     7  .white 00;37
     8  .redb 01;31
     9  .greenb 01;32
```

## 3) less/more

Are finding that "cat-ing" a file is causing your terminal to scroll too fast? Use `less` to fix that problem. But wait, what about `more`? `less` is actually based on `more`. Early versions of `more` were unable to scroll backward through a file. In any case, `less` has a nice ability to scroll through the contents of a file or output with space/down/up/page keys. Use `q` to exit.

* Need line numbers? use `less -N`
* Need to search while in less? Use `/wordshere` to search.
* Once you're searching use `n` to go to the next result, and `N` for the previous result.
* Want to open up less with search already? Use `less -pwordshere file.txt`
* Whitespace bothering you? `less -s`
* Multiple files, what!? `less file1.txt file2.txt`
* Next file is `:` and then hit `n`, previous file is `:` then hit `p`
* Need to pipe some log output or the results of a curl? Use `curl dev.to | less`
* Less has bookmarks? Yep. Drop a marker in less for the current top line with `m` then hit any letter as the bookmark like `a`. Then to go back, hit the `'` (apostrophe key) and the bookmark letter to return (in this case `a`).
* Want to drop into your default editor from less right where you are currently at and return back when you're done? Use `v` and your default terminal editor will open up at the right spot. Then once you've quit/saved in that editor you will be right back where you were before. 😎 Awesome! 🎉

## 4) [curl](https://curl.haxx.se/docs/)

Curl is another essential tool if you need to do just about any type of protocol request. Here is a small fraction of ways you can interact with curl.

* *GET* `curl https://dev.to/
* *Output to a file* `curl -o output.html https://dev.to/`
* *POST* `curl -X POST -H "Content-Type: application/json" -d '{"name":"tom"}' http://localhost:8080`
* *BASIC AUTH* `curl -u username:password http://localhost:8080
* *drop into cat* `cat | curl -H 'Content-Type: application/json' http://localhost:8080 -d @-`
* *HEAD* `curl -I dev.to`
* *Follow Redirects* `curl -I -L dev.to`
* *Pass a certificate, skip verify* `curl --cert {{client.pem}} --key {{key.pem}} --insecure https://example.com`

## 5) man

If you are stuck understanding what a command does, or need the documentation. Use `man`! It's literally the `manual` and it works on just all the built in commands. It even works on itself

```
$ man man
NAME
       man - format and display the on-line manual pages

SYNOPSIS
       man [-acdfFhkKtwW] [--path] [-m system] [-p string] [-C config_file] [-M pathlist] [-P pager] [-B browser] [-H htmlpager] [-S section_list] [section] name ...

DESCRIPTION
       man  formats  and displays the on-line manual pages.  If you specify section, man only looks in that section of the manual.  name is normally the name of the manual page, which is typically the name of a com-
       mand, function, or file.  However, if name contains a slash (/) then man interprets it as a file specification, so that you can do man ./foo.5 or even man /cd/foo/bar.1.gz.

       See below for a description of where man looks for the manual page files.

MANUAL SECTIONS
       The standard sections of the manual include:

       1      User Commands

       2      System Calls

       3      C Library Functions

       4      Devices and Special Files

       5      File Formats and Conventions

       6      Games et. Al.

       7      Miscellanea

       8      System Administration tools and Deamons

       Distributions customize the manual section to their specifics, which often include additional sections.
```

## 6) alias

If you ever need to setup a short command name to execute a script or some complicated `git` command for instance, then use `alias`.

```
alias st="git status"
alias branches="git branch -a --no-merged"
alias imgcat="~/dotfiles/imgcat.sh"
```

Bonus: add these to your `~/.bashrc` to execute these when your shell starts up!

## 7) echo

The "hello world" of any terminal is `echo`. `echo "hello world"`. You can include environment variables and even sub commands `echo "Hello $USER"`.

## 8) [sed](https://www.gnu.org/software/sed/manual/sed.html)

Sed is a "stream editor". This means we can do a number of different ways to read some input, modify it, and output it. Here's a few ways you can do that:

* Text substitutions: `echo 'Hello world!' | sed s/world/tom/`
* Selections: `sed -n '1,4p' _bashrc` (`n` is quiet or suppress unmatched lines, `1,4p` is `p` print lines 1-4.
* Multiple selections: `sed -n -e '1,4p' -e '8-10p' _bashrc`
* Every X lines: `sed -n 1~2p _bashrc` (use `~` instead of `,` to denote every 2 lines (in this case 2)
* Search all/replace all: `sed s/world/tom/gp` (`g` for global search, `p` is to print each match instance)

**NOTE** the sed implementation might differ depending on the system you are using. Keep this in mind that some flags might be unavailable. Take a look at the `man sed` for more info.

## 9) [tar](https://www.gnu.org/software/tar/manual/tar.html)

![tar xkcd](https://dev-to-uploads.s3.amazonaws.com/i/nzg4dtdu2stmewluibmh.png)

If you need to create an archive of a number of files. Don't worry, you'll remember these flags soon enough!

* `-c` **create**
* `-v` **verbose**
* `-f` **file name**

This would look like:

```
tar -cvf archive.tar files/
```

By default, tar will create an uncompressed archive unless you tell it to use a specific compression algorithm.

* `-z` **gzip** (decent compression, reasonable speed) (.gz)
* `-j` **bzip2** (better compression, slower) (*.bz2)

```
tar -cvfz archive.tar.gz files/
```

Extraction is done with:

* `-x` **extract**

Similar options for decompression options and verbose:

```
# extract, verbose, gzip decompress, filename
tar -xvzf archive.tar.gz
```

## 10) cd

Change directories. Not much to it! `cd ../../`, `cd ~` `cd files/` `cd $HOME`

## 11) head

> `head [-n count | -c bytes] [file ...]`
> `head` is a filter command that will display the first `count` (`-c`) lines or `bytes` (`-b`) of each of the specified files, or of the standard input if no files are specified. If `count` is omitted, it defaults to 10.

> If more than a single file is specified, each file is preceded by a header consisting of the string ''==> XXX <=='' where XXX is the name of the file.

```
head -n 10 ~/dotfiles/_bashrc`
```

## 12) mkdir

> mkdir creates the directories named as operands, in the order specified, using mode rwxrwxrwx (0777) as modified by the current umask. With the following modes:
> * -m mode: Set the file permission bits of the final directory created (mode can be in the format specified by `chmod`
> * -p Create intermediary directories as required (if not specified then the full path prefix must already exist)
> * -v Verbose when creating directories

## 13) rm

> remove the non-directory type files specified on the command line. If permissions of the file do not permit writing, and standard input is terminal, user is prompted for confirmation.

A few options of rm can be quite useful:

* `-d` remove directories as well as files
* `-r` `-R` Remove the file hierarchy rooted in each file, this implies the `-d` option.
* `-i` Request confirmation before attempting to remove each file
* `-f` Remove files without prompting confirmation regardless of permissions. Do not display diagnostic messages if there are errors or it doesn't exist
* `-v` verbose output

## 14) mv

> mv renames the file named by the source to the destination path. mv also moves each file named by a source to the destination. Use the following options:

* `-f` Do not prompt confirmation for overwriting
* `-i` Cause mv to write a prompt to stderr before moving a file that would overwrite an existing file
* `-n` Do not overwrite an existing file (overrides `-i` and `-f`)
* `-v` verbose output

## 15) cp

> copy the contents of the source to the target

* `-L` sumbolic links are followed
* `-P` Default is no symbolic links are followed
* `-R` if source designates a directory, copy the directory and entire subtree (created directories have the same mode as the source directory, unmodified by the process umask)

## 16) ps

> Display the header line, followed by lines containing information about all of your processes that have controlling terminals. Various options can be used to control what is displayed.

* `-A` Display information about other users' processes
* `-a` Display information about other users' processes as well as your own (skip any processes without controlling terminal)
* `-c` Change command column to contain just exec name
* `-f` Display uid, pid, parent pid, recent CPU, start time, tty, elapsed CPU, command. `-u` will display user name instead of uid.
* `-h` Repeat header as often as necessary (one per page)
* `-p` Display about processes which match specified process IDS (`ps -p 8040`)
* `-u` Display belonging to the specified user (`ps -u tom`)
* `-r` Sort by CPU usage

```
 UID   PID  PPID   C STIME   TTY           TIME CMD                     F PRI NI       SZ    RSS WCHAN     S             ADDR
  501 97993 78315   0  5:28PM ??       134:30.10 Figma Beta Helpe     4004  31  0 28675292 316556 -      R                   0
   88   292     1   0 14Aug20 ??       372:58.39 WindowServer         410c  79  0  8077052  81984 -      Ss                  0
  501 78315     1   0 Thu04PM ??        17:55.75 Figma Beta        1004084  46  0  5727912 109596 -      S                   0
  501 78377 78315   0 Thu04PM ??        22:16.66 Figma Beta Helpe     4004  31  0  5893304  59376 -      S                   0
  501 70984 70915   0 Wed02PM ??         8:58.36 Spotify Helper (     4004  31  0  9149416 294276 -      S                   0
  202   266     1   0 14Aug20 ??       108:51.87 coreaudiod           4004  97  0  4394220   6960 -      Ss                  0
  501 70979 70915   0 Wed02PM ??         2:09.53 Spotify Helper (     4004  31  0  4767800  49764 -      S                   0
  501 97869 78315   0  5:28PM ??         0:32.51 Figma Beta Helpe     4004  31  0  5324624  81000 -      S                   0
  501 70915     1   0 Wed02PM ??         9:53.82 Spotify           10040c4  97  0  5382856  92580 -      S                   0
```

## 17) [tail](https://en.wikipedia.org/wiki/Tail_(Unix))

Similar to `head`, tail will display the contents of a file or input starting at the given options:

* `tail -f /var/log/web.log` Commonly used to not stop the output when the end of the file is reached, but wait for additional data to be appended. (Use `-F` to follow when the file has been renamed or rotated)
* `tail -n 100 /var/log/web.log` Number of lines
* `tail -r` Input is displayed in reverse order
* `tail -b 100` Use number of bytes instead of lines

## 18) kill

> Send a signal to the processes specified by the pid

Commonly used signals are among:

```
     1       HUP (hang up)
     2       INT (interrupt)
     3       QUIT (quit)
     6       ABRT (abort)
     9       KILL (non-catchable, non-ignorable kill)
     14      ALRM (alarm clock)
     15      TERM (software termination signal)

```

You will typically see a `kill -9 pid`. Find out the process with `ps` or `top`!

## 19) top

Need some realtime display of the running processes? Use `top` for this!

```
Processes: 517 total, 3 running, 3 stuck, 511 sleeping, 3013 threads                                                                                                                       16:16:07
Load Avg: 2.54, 2.63, 2.57  CPU usage: 12.50% user, 5.66% sys, 81.83% idle  SharedLibs: 210M resident, 47M data, 17M linkedit.
MemRegions: 153322 total, 5523M resident, 164M private, 2621M shared. PhysMem: 16G used (2948M wired), 431M unused.
VM: 2539G vsize, 1995M framework vsize, 14732095(0) swapins, 17624720(0) swapouts. Networks: packets: 81107619/74G in, 103172624/63G out. Disks: 44557301/463G read, 15432059/228G written.

PID    COMMAND      %CPU TIME     #TH   #WQ  #PORTS MEM    PURG   CMPRS  PGRP  PPID  STATE    BOOSTS          %CPU_ME %CPU_OTHRS UID  FAULTS    COW       MSGSENT     MSGRECV     SYSBSD
97993  Figma Beta H 53.0 02:19:46 26    1    271    347M+  0B     109M   78315 78315 sleeping *0[1]           0.00000 0.00000    501  5042481+  5175      29897392+   8417371+    19506598+
62329  Slack Helper 21.6 05:18.63 20    1    165+   123M-  0B     27M    62322 62322 sleeping *0[4]           0.00000 0.00000    501  2124802+  13816     813744+     435614+     1492014+
0      kernel_task  9.6  07:47:25 263/8 0    0      106M   0B     0B     0     0     running   0[0]           0.00000 0.00000    0    559072    0         1115136682+ 1057488639+ 0
60459  top          5.5  00:00.65 1/1   0    25     5544K+ 0B     0B     60459 83119 running  *0[1]           0.00000 0.00000    0    3853+     104       406329+     203153+     8800+
```

## 20) and 21) [chmod](https://en.wikipedia.org/wiki/Chmod), [chown](https://www.man7.org/linux/man-pages/man1/chown.1.html)

File permissions are likely a very typical issue you will run into. Judging from the number of results for "permission not allowed" and other variations, it would be very useful to understand these two commands when used in conjunction with one another.

When you list files out, the permission flags will denote things like:

```
-rwxrwxrwx
```

`-` denotes a file, while `d` denotes a directory. Each part of the next three character sets is the actual permissions. 1) file permissions of the owner, 2) file permissions of the group, 3) file permissions for others. **r** is read, **w** is write, **x** is execute.

Typically, `chmod` will be used with the numeric version of these permissions as follows:

```
    0: No permission
    1: Execute permission
    2: Write permission
    3: Write and execute permissions
    4: Read permission
    5: Read and execute permissions
    6: Read and write permissions
    7: Read, write and execute permissions
```

So if you wanted to give read/write/execute to owner, but only read permissions to the group and others it would be:

```
chmod 744 file.txt
```

With `chown` you can change the owner and the group of a file as such as `chown $USER: file.txt` (to change the user to your current user and to use the default group).

## 22) grep

Grep lets you search on any given input, selecting lines that match various patterns. Usually grep is used for simple patterns and basic regular expressions. `egrep` is typically used for extended regex.

If you specify the `--color` this will highlight the output. Combine with `-n` to include numbers.

```
grep --color -n "imgcat" ~/dotfiles/_bashrc
251:alias imgcat='~/dotfiles/imgcat.sh'
```

## 23) find

Recursively descend the directory tree for each path listed and evaluate an expression. Find has a lot of variations and options, but don't let that scare you. The most typical usage might be:

* `find . -name "*.c" -print` print out files where the name ends with .c
* `find . \! -name "*.c" -print` print out files where the name **does not** end in .c
* `find . -type f -name "test" -print` print out only type files (no directories) that start with the name "test"
* `find . -name "*.c" -maxdepth 2` only descend 2 levels deep in the directories

## 24) ping

Among **many** network diagnostic tools from [lsof](https://dev.to/nyxtom/why-is-port-8080-already-in-use-and-what-else-is-running-using-lsof-to-navigate-your-system-567e) to `nc`, you can't go wrong with `ping`. Ping simply sends `ICMP` request packets to network hosts. Many servers disable ICMP responses, but in any case, you can use it in a number of useful ways.

* Specify a time-to-live with `-T`
* Timeouts `-t`
* `-c` Stop sending and receiving after **count** packets.
* `-s` Specify the number of data bytes to send

```
PING dev.to (151.101.130.217): 56 data bytes
64 bytes from 151.101.130.217: icmp_seq=0 ttl=58 time=17.338 ms
64 bytes from 151.101.130.217: icmp_seq=1 ttl=58 time=32.732 ms
64 bytes from 151.101.130.217: icmp_seq=2 ttl=58 time=14.288 ms
64 bytes from 151.101.130.217: icmp_seq=3 ttl=58 time=15.166 ms
64 bytes from 151.101.130.217: icmp_seq=4 ttl=58 time=16.465 ms
--- dev.to ping statistics ---
5 packets transmitted, 5 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 14.288/19.198/32.732/6.848 ms
```

## 25) sudo

![sudo xkcd](https://dev-to-uploads.s3.amazonaws.com/i/daobnxyaeprx01avl3mf.png)

This command is required if you want to do actions that require the root or superuser or another user as specified by the security policy.

```
sudo ls /usr/local/protected
```

## Conclusion

There are *a lot* of really useful commands available. I simply could not list them all out here without doing them a disservice. I would add to this list a number of *very important* utilities like `df`, `free`, `nc`, `lsof`, and loads of other diagnostic commands. Not to mention, many of these commands actually deserve their own post
