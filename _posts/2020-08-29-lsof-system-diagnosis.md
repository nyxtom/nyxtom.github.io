---
title: lsof to diagnose network and system issues
published: true
description: How to diagnose network and system issues with lsof
tags: [linux,productivity,beginners,bash]
---

Linux has a plethora of great built in command line tools. Skilled system administrators can do most of their work with these tools without necessarily having to write a lot of scripts or install additional packages. If you've ever run into an error:

```
Port 8080 is already in use
```

You may be surprised to know that you can resolve this quite quickly with the `lsof` command to find out what's running on your system. lsof is a fantastic tool to use for finding out about files opened by processes, because in Unix everything is a file.

## lsof

Since everything is a file, you can think of a socket as a file that writes to the network interface. Here are a few handy examples to use:

### List all listening TCP ports

```
lsof -nP -iTCP -sTCP:LISTEN
```

There are so many options to use with lsof, the ones I've listen in this command are:

* `-n` Do not convert port numbers to port names
* `-P` Do not resolve hostnames, show numerical addresses
* `-iTCP` Show only network TCP files
* `-sTCP:LISTEN` Select protocol states (show only TCP state LISTEN)

```
COMMAND   PID     USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
sshd      445     root    3u  IPv4  16434      0t0  TCP *:22 (LISTEN)
sshd      445     root    4u  IPv6  16445      0t0  TCP *:22 (LISTEN)
apache2   515     root    4u  IPv6  16590      0t0  TCP *:80 (LISTEN)
mysqld    534    mysql   30u  IPv6  17636      0t0  TCP *:3306 (LISTEN)
mysqld    534    mysql   33u  IPv6  19973      0t0  TCP *:33060 (LISTEN)
apache2   764 www-data    4u  IPv6  16590      0t0  TCP *:80 (LISTEN)
apache2   765 www-data    4u  IPv6  16590      0t0  TCP *:80 (LISTEN)
master    929     root   13u  IPv4  19637      0t0  TCP *:25 (LISTEN)
master    929     root   14u  IPv6  19638      0t0  TCP *:25 (LISTEN)
```

### List processes listening on a specific port

```
lsof -nP -iTCP:8080
```

* `-i` select by IPv[46] address: [46][proto][@host|addr][:svc_list|port]

Note that you can specify all of the above options for a more complete analysis when selecting socket files.

```
COMMAND     PID   USER   FD   TYPE            DEVICE SIZE/OFF NODE NAME
nginx      4692 nyxtom    6u  IPv4 0x8a060f1f95180c3      0t0  TCP *:8080 (LISTEN)
nginx      4837 nyxtom    6u  IPv4 0x8a060f1f95180c3      0t0  TCP *:8080 (LISTEN)
```

The below example will select all IPv4 files at the *127.0.0.1* host and port *27017*

```
lsof -i4@127.0.0.1:27017
```

```
COMMAND  PID   USER   FD   TYPE            DEVICE SIZE/OFF NODE NAME
mongod  4680 nyxtom   10u  IPv4 0x8a060f1f8ea5223      0t0  TCP 127.0.0.1:27017 (LISTEN)
```

### Listing open files held by processes executing a command

* `lsof -c` List all open files file held by processes executing a command that begins with the characters after `c`.

For example, the following will list all open files held by processes that have executed the node command.

```
lsof -cnode
```

```
node    4905 nyxtom   21u     unix  0x8a060f1fdfabffb      0t0                     ->0x8a060f1fdfac0c3
node    4905 nyxtom   22u     unix  0x8a060f1fdfac253      0t0                     ->0x8a060f1fdfab82b
node    4905 nyxtom   23w      REG                1,4     4965          8705157713 /Users/nyxtom/Library/Logs/NGLClient_CCLibrary13.8.log
node    4905 nyxtom   24u      REG                1,4    53248          8705157717 /Users/nyxtom/Library/Caches/node/Cache.db
node    4905 nyxtom   25u      REG                1,4        0          8705157720 /Users/nyxtom/Library/Caches/node/Cache.db-wal
node    4905 nyxtom   26r   PSXSHM                        4096                     FNetwork.defaultStorageSession
node    4905 nyxtom   27u      REG                1,4    32768          8705157721 /Users/nyxtom/Library/Caches/node/Cache.db-shm
node    4905 nyxtom   28   NPOLICY
node    4905 nyxtom   29r      CHR                3,2      0t0                 313 /dev/null
node    4905 nyxtom   30u    systm  0x8a060f1f09b2ed3      0t0                     [ctl com.apple.netsrc id 7 unit 34]
node    4905 nyxtom   31u    systm  0x8a060f1ed12b7b3      0t0                     [ctl com.apple.netsrc id 7 unit 33]
node    4905 nyxtom   33u     IPv4  0x8a060f1fe986e63      0t0                 TCP localhost:49540 (LISTEN)
node    4905 nyxtom   34u     unix  0x8a060f1fdfab69b      0t0                     ->0x8a060f1fdfac573
node    4905 nyxtom   36u     IPv4  0x8a060f1fe987843      0t0                 TCP localhost:49541 (LISTEN)
node    4905 nyxtom   37u     IPv4  0x8a060f1fe985aa3      0t0                 TCP localhost:45623 (LISTEN)
node    4905 nyxtom   38u     IPv4  0x8a060f1fe9850c3      0t0                 TCP localhost:59466 (LISTEN)
```

### Listing open files held by a specific process id

If you need to figure out open files held by a specific process use the `-p` flag. This one is one of the most useful flags I've used with `lsof`. If I'm ever running into a problem with a specific process, this flag can help me figure out what in the world that process is doing and who it is talking to and where.

```
lsof -p 4905
```

```
COMMAND   PID   USER   FD      TYPE             DEVICE  SIZE/OFF                NODE NAME
Spotify 70915 nyxtom  cwd       DIR                1,4       736                   2 /
Spotify 70915 nyxtom  txt       REG                1,4  30207680          8711314045 /Applications/Spotify.app/Contents/MacOS/Spotify
Spotify 70915 nyxtom  txt       REG                1,4 143063984          8711313410 /Applications/Spotify.app/Contents/Frameworks/Chromium Embedded Framework.framework/Chromium Embedded Framework
Spotify 70915 nyxtom  txt       REG                1,4     28420          8712403338 /Library/Preferences/Logging/.plist-cache.vmIBl3ut
Spotify 70915 nyxtom  txt       REG                1,4  28484912 1152921500312402062 /usr/share/icu/icudt64l.dat
Spotify 70915 nyxtom  txt       REG                1,4    240192          8699253542 /private/var/db/timezone/tz/2020a.1.0/icutz/icutz44l.dat
Spotify 70915 nyxtom  txt       REG                1,4   1568368 1152921500312400600 /usr/lib/dyld
Spotify 70915 nyxtom  txt       REG                1,4   2353284 1152921500312081779 /System/Library/Fonts/Helvetica.ttc
Spotify 70915 nyxtom  txt       REG                1,4    228344 1152921500312412892 /System/Library/CoreServices/SystemAppearance.bundle/Contents/Resources/FunctionRowAppearance.car
Spotify 70915 nyxtom  txt       REG                1,4     10139          8711313924 /Applications/Spotify.app/Contents/Resources/Touchbar Back.pdf
Spotify 70915 nyxtom  txt       REG                1,4     10182          8711313886 /Applications/Spotify.app/Contents/Resources/Touchbar Forward.pdf
Spotify 70915 nyxtom  txt       REG                1,4     12829          8711313905 /Applications/Spotify.app/Contents/Resources/Touchbar Search.pdf
Spotify 70915 nyxtom  txt       REG                1,4     12760          8711313885 /Applications/Spotify.app/Contents/Resources/Touchbar Shuffle.pdf
```

### Logical AND/OR/NEGATE operations with lsof

Normally the flags will combine with `OR`. But you can also use the `-a` to combine results as a logical `AND` or you can use the `^` to negate results.

```
# find all the open tcp listening sockets held by the $USER
lsof -iTCP -sTCP:LISTEN -u $USER -a
```

## Conclusion

There are loads of useful built in linux command line tools you can use to debug your system. `lsof` just happens to be one of my favorite utilities. For more details you can type `man lsof` for all the documentation and many other examples.
