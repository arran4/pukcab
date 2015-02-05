# Pukcab

## Features
 * lightweight (just 1 binary to be installed on both the client and the server)
 * easy to install (only 1 username with SSH connectivity is required to set up a server)
 * flexible configuration
 * sensible defaults
 * automatic retention schedules
 * incremental/full backups
 * data de-duplication
 * data compression

## Requirements
### Backup server
 * SSH server
 * dedicated user (recommended)
 * disk space

### Clients
 * SSH client
 * functional `tar` command (tested with [GNU tar](http://www.gnu.org/software/tar/), should work with [BSD (libarchive) tar](http://www.libarchive.org/) and [Jörg Schilling's star](http://sourceforge.net/projects/s-tar/))

## Installation
Just copy the `pukcab` binary into your path (`/usr/bin/pukcab` will be just fine) on the backup server and each client.

On the backup server

1. create a dedicated user -- this user does not need specific privileges (i.e. do NOT use `root`)
2. allow key-based SSH login for that user (**mandatory**)
2. *optional*: allow password-based SSH login and set a password for the dedicated user (if you want to be able to register new clients using that password)

On the clients (if not using a password to register clients)

1. [create SSH keys][] for the user which will launch the backup (most probably `root`)
1. add the user's public key to the dedicated user's `authorized_keys` on the backup server

On the clients (if using a password to register clients)

1. [create SSH keys][] for the user which will launch the backup (most probably `root`)
1. [register][] to the backup server

## Configuration
`pukcab` is configured with a simple [INI-like text file](https://en.wikipedia.org/wiki/INI_file):

```
; comment
name1 = number
name2 = "text value"
name3 = [ "list", "of", "text", "values" ]
...
```
Notes

 * text values must be enclosed in `"`
 * lists of values are enclosed in `[` and `]` with comma-separated items

The default is to read `/etc/pukcab.conf` then `~/.pukcabrc` (which means that this user-defined file can override values set in the global configuration file).

Both client and server use the same configuration file format and location, only the values determine the client or server role (a client will have a `server` parameter set).

### Server
parameter | type | default | description
----------|------|---------|-------------------------------
`user`    | text | none    | **mandatory** specifies the user name `pukcab` will run under
`vault`   | text |`"vault"`| specifies the folder where all archive files will be created
`catalog` | text |`"catalog.db"`| specifies the name of the catalog database
`maxtries`|number| `10`    | number of retries in case of concurrent client accesses

Notes

 * `vault` and `catalog` paths can be absolute (starting with `/`) or relative to `user`'s home directory.
 * the `vault` folder must be able to store many gigabytes of data, spread over thousands of files
 * the `catalog` database may become big and must be located in a folder where `user` has write access
 * the `vault` folder **must not be used to store anything** else than `pukcab`'s data files; in particular, do **NOT** store the `catalog` there

Example:

```
; all backups will be received and stored by the 'backup' user
user="backup"
vault="/var/local/backup/vault"
catalog="/var/local/backup/catalog.db"
```

### Client
parameter | type | default   | description
----------|------|-----------|-------------------------------
`user`    | text | none      | **mandatory** specifies the user name `pukcab` will use to connect to the backup server
`server`  | text | none      | **mandatory** specifies the backup server to connect to
`port`    |number| 22        | specifies the TCP port to use to connect to the backup server
`include` | list | cf. below | specifies filesystems/folders/files to include in the backup
`exclude` | list | cf. below | specifies filesystems/folders/files to exclude from the backup

Some default values depend on the platform:

parameter | Linux
----------|------------------------------------------------------------
`include` | `[ "ext2", "ext3", "ext4", "btrfs", "xfs", "jfs", "vfat" ]`
`exclude` | `[ "/proc", "/sys", "/selinux", "tmpfs", "./.nobackup" ]`

Example:

```
user="backup"
server="backupserver.localdomain.net"
```

## Usage
### Synopsis
> `pukcab` _[COMMAND](#commands)_ [ _[OPTIONS](#options)_... ] [ _[FILES](#files)_... ]

#### Commands
**backup and recovery**   | .
--------------------------|---------------------------------------
[backup][], [save][]      | take a new backup
[continue][], [resume][]  | continue a partial backup
[restore][]               | restore files
[verify][], [check][]     | verify files in a backup
**maintenance** | 
[delete][], [purge][]     | delete a backup
[expire][]                | apply retention schedule to old backups
**utilities** |
[info][], [list][]        | list backups and files
[ping][], [test][]        | check server connectivity
[register][]              | register to backup server

### backup
This command launches a new backup:

 * creates a new backup set (and the corresponding date/id) on the [backup server](#server)
 * builds the list of files to be backed-up, based on the `include`/`exclude` configuration directives
 * sends that list to the backup server
 * computes the list of changes since the last backup (if `--full` isn't specified)
 * sends the files to be includes in the backup
 * closes the backup
Interrupted backups can be resumed with the [#continue continue] command

Syntax
>  `pukcab backup` [ --[full][] ] [ --[name][]=_name_ ] [ --[schedule][]=_schedule_ ]

Note

 * the [name][] and [schedule][] options are chosen automatically if not specified

### continue
This command continues a previously interrupted backup.

Syntax
>  `pukcab continue` [ --[name][]=_name_ ] [ --[date][]=_date_ ]

Notes

 * the [name][] option is chosen automatically if not specified
 * the [date][] option automatically selects the last unfinished backup
 * only unfinished backups may be resumed

### restore
This command restores [files][] as they were at a given [date][].

Syntax
>  `pukcab restore` [ --[name][]=_name_ ] [ --[date][]=_date_ ] [ _[FILES](#files)_... ]

Notes

 * the [name][] option is chosen automatically if not specified
 * the [date][] option automatically selects the last backup
 * this operation currently requires a working `tar` system command (usually GNU tar)
 
### verify
This command reports [files][] which have changed since a given [date][].

Syntax
>  `pukcab verify` [ --[name][]=_name_ ] [ --[date][]=_date_ ] [ _[FILES](#files)_... ]

Notes

 * the [name][] option is chosen automatically if not specified
 * the [date][] option automatically selects the last backup if not specified
 
### delete
This command discards the backup taken at a given [date][].

Syntax
>  `pukcab delete` [ --[name][]=_name_ ] --[date][]=_date_

Notes

 * the [name][] option is chosen automatically if not specified
 * the [date][] must be specified
 
### expire
This command discards backups following a given [schedule][] which are older than a given [age (or date)](#date). Standard retention schedules have pre-defined retention periods:

 schedule   | retention period
------------|------------------
 `daily`    | 2 weeks
 `weekly`   | 6 weeks
 `monthly`  | 365 days
 `yearly`   | 10 years

Syntax
>  `pukcab expire` [ --[name][]=_name_ ] [ --[schedule][]=_schedule_ ] [ --[age][]=_age_ ] [ --[keep][]=_keep_ ]

Notes

 * on the [backup server](#server), the [name][] option defaults to all backups if not specified
 * on a [backup client](#client), the [name][] option is chosen automatically if not specified
 * the [schedule][] and [expiration][] are chosen automatically if not specified
 
### info
This command lists the backup sets stored on the server. Backup sets can be filtered by name and/or date and files.

Syntax
>  `pukcab info` [ --[name][]=_name_ ] [ --[date][]=_date_ ] [ _[FILES](#files)_... ]

Notes

 * if [date][] is specified, the command lists only details about the corresponding backup
 * if [name][] is not specified, the command lists all backups, regardless of their name
 * verbose mode lists the individual [files][]

### ping
This command allows to check connectivity to the server.

Syntax
>  `pukcab ping`

Notes

 * verbose mode displays detailed information during the check

### register
This command registers a client's SSH public key to the server.

Syntax
>  `pukcab register`

Notes

 * to register to the backup server, `pukcab` will ask for the dedicated user's password (set on the server)
 * verbose mode displays detailed information during the registration

### Options
`pukcab` is quite flexible with the way options are provided:

 * options can be provided in any order
 * options have both a long and a short (1-letter) name (for example, `--name` is `-n`)
 * options can be prefixed with 1 or 2 minus signs (`--option` and `-option` are equivalent)
 * `--option=value` and `--option value` are equivalent (caution: `=` is mandatory for boolean options)

This means that the following lines are all equivalent:

> `pukcab info -n test`

> `pukcab info -n=test`

> `pukcab info --n test`

> `pukcab info --n=test`

> `pukcab info -name test`

> `pukcab info -name=test`

> `pukcab info --name test`

> `pukcab info --name=test`

#### General options
The following options apply to all commands:

option                    | description
--------------------------|------------------------------------------------
`-c`, `--config`[=]_file_ | specify a [configuration file](#configuration) to use
`-v`, `--verbose`[`=true`]| display more detailed information
`-h`, `--help`            | display online help

#### date
Dates are an important concept for `pukcab`.

All backup sets are identified by a unique numeric id and correspond to a set of files at a given point in time (the backup id is actually a [UNIX timestamp](https://en.wikipedia.org/wiki/Unix_time)).
The numeric id can be used to unambiguously specify a given backup set but other, more user-friendly formats are available:

 * a duration in days (default when no unit is specified), hours, minutes is interpreted as relative (in the past, regardless of the actual sign of the duration you specify) to the current time
 * a human-readable date specification in YYYY-MM-DD format is interpreted as absolute (00:00:00 on that date)
 * `now` or `latest` are interpreted as the current time
 * `today` is interpreted as the beginning of the day (local time)

Syntax
>  `--date`[=]*date*

>  `-d` *date*

Examples

 > `--date 1422577319` means *on the 30th January 2015 at 01:21:59 CET*
 
 > `--date 0`, `--date now` and `--date latest` mean *now*
 
 > `--date today` means *today at 00:00*
 
 > `--date 1` means *yesterday same time*
 
 > `--date 7` means *last week*
 
 > `--date -48h` and `--date 48h` both mean *2 days ago*
 
 > `--date 2h30m` means *2 hours and 30 minutes ago*
 
 > `--date 2015-01-07` means *on the 7th January 2015 at midnight*

#### name
In `pukcab`, a name is associated with each backup when it's created. It is a free-form text string.

Syntax
>  `--name`[=]*name*

>  `-n` *name*

Default value
>  current host name (output of the `hostname` command)

#### schedule
In `pukcab`, a retention schedule is associated with each backup when it's created and is used when expiring old backups. It is a free-form text string but common values include `daily`, `weekly`, `monthly`, etc.

Syntax
>  `--schedule`[=]*schedule*

>  `-r` *schedule*

Default value (the default value depends on the current day)

 * `daily` from Monday to Saturday
 * `weekly` on Sunday
 * `monthly` on the 1st of the month
 * `yearly` on 1st January

#### full
Forces a full backup: `pukcab` will send all files to the server, without checking for changes.

Syntax
>  `--full`[`=true`]

>  `--full=false`

>  `-f`

Default value
>  `false`

#### keep
When expiring data, keep at least a certain number of backups (even if they are expired).

Syntax
>  `--keep`[=]*number*

>  `-k` *number*

Default value
>  `3`


#### files
File names can be specified using the usual shell-like wildcards `*` (matches any number of characters) and `?` (matches exactly one character). The following conventions apply:

 * file names starting with a slash ('`/`') are absolute
 * file names not starting with a slash ('`/`') are relative
 * specifying a directory name also selects all the files underneath

Examples

>  `/home` includes `/home/john`, `/home/dave`, etc. and all the files they contain (i.e. all users' home directories)

>  `*.jpg` includes all `.jpg` files (JPEG images) in all directories

>  `/etc/yum.repos.d/*.repo` includes all repositories configured in Yum[[br]]

>  `/lib` includes `/lib` and all the files underneath but not `/usr/lib`, `/var/lib`, etc.
 

## Examples
### Launch a new backup - default options
```
[root@myserver ~]# pukcab backup --verbose
Starting backup: name="myserver" schedule="daily"
Sending file list... done.
New backup: date=1422549975 files=309733
Previous backup: date=1422505656
Determining files to backup... done.
Incremental backup: date=1422549975 files=35
Sending files... done
[root@myserver ~]#
```

[backup]: #backup
[continue]: #continue
[resume]: #continue
[save]: #backup
[restore]: #restore
[verify]: #verify
[check]: #verify
[delete]: #delete
[purge]: #delete
[expire]: #expire
[info]: #info
[list]: #info
[ping]: #ping
[test]: #ping
[register]: #register

[name]: #name
[date]: #date
[schedule]: #schedule
[full]: #full
[keep]: #keep
[files]: #files
[age]: #date

[create SSH keys]: https://en.wikipedia.org/wiki/Ssh-keygen