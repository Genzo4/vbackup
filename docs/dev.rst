=================
Extending vbackup
=================

If you're interested in extending vbackup then read below and have a look
at the existing modules (I recommend mount and tar/xfsdump).
A template for new modules is available with the name 'skel' under the
scripts/ directory of the source code.

If you believe you've created something that will be useful for others
too, I'll be glad to include it in the distribution (preserving your
copyright of course)


File tree
=========

The installation layout should comply with the latest FHS (2.3 as of Nov 2006).
Everything different than that should be reported as bug.

::

 ${bindir}/			The vbackup program

 ${sysconfdir}/
 	vbackup/
 		rc.d/		Backup configurations
 		backup.daily/	Links to appropriate rc.d scripts
 			vbackup.conf	Per backup global configuration file
 		backup.weekly/	....
 		backup.monthly/	....

 ${datadir}/
 	vbackup/
 		bin/		Core scripts (run) used by vbackup
 				(They are standalone scripts)
 		helpers/	Helper scripts (not standalone)
 		scripts/	The backup scripts
 		wizard/		Wizard configuration files
 		samples/	Sample configurations. Also used for --rc
 
 ${docdir}/
 	vbackup/		Documentation


Backup scripts
==============

Each backup script must provide:

Functions
---------

do_help()
        A function to display help and return

do_check_conf()
        Check whether all configuration variables are ok

        Return: 0: OK, 1: Error

        May display error messages

do_run()
        Do the backup

Variables
---------

=============== ===========================================================
NAME            The script name ("psql")
VERSION         The script version ("1.0.0")
DESC            A short description ("Backup a postgresql database")
COPYRIGHT       Copyright ("Copyright (c) 2006 Harhalakis Stefanos")
LICENCE         The license of the script ("GPLv2")
CONTACT         A contact email for bugreports etc ("v13@v13.gr")
=============== ===========================================================

It may expect
-------------

  * If the configuration file includes a DESTDIR options, it will be
    prepended with the DESTDIR0 prefix. It will also be transformed
    by changing %X% variables (for example %D1%)
  * A variable named ABORT is set to 1 if a script previously exited with
    error code 2 (see below). Most scripts should abort with exit code 0
    when they detect this. For example: a script that checks for free space
    may exit with error code 2. The ABORT variable will be set automatically.
    All backup scripts must detect this and do nothing. The umount script
    will ignore this and actually try to umount the directory.
  * A variable named LEVEL that indicates the backup level as passed to the
    vbackup command.
  * A variable named STRATEGY that indicates the strategy as passed to the
    vbackup command. This will be empty if no strategy was specified.
  * A variable named STATEDIR that points to a script-specific directory
    to store state. Note that the directory may not exist. If needed, the
    script should use h_ensuredir to create the directory

It must return (from each function)
-----------------------------------

======= ====================================================
0	Everything were OK
1	Error occurred - continue backups
2	Error occurred - abort backup sequence
3	Error occurred - force abort of backup sequence
======= ====================================================

Notes:

  * When a script returns 2, the flag ABORT is set to 1. This is
    checked by backup scripts and they will not do anything at all.
    Scripts like umount may ignore this and umount the directory
  * When a script returns 3, the backup is immediately aborted.
    This should be used by scripts like mount

Configuration files:
====================

Some configuration files may include a DESTDIR directive.
For example, '%D1%' which will be replaced by YYMMDD of the current date
E.g.: /tmp/backups/filesystem/%D1%
will be converted to: /tmp/backups/filesystem/061117 (as of 17 Nov 2006)

The following special sequences are recognised:

======= ======================================
%D0%	Day of the week (1-7). 1 is Monday
%D1%	YYMMDD of the current date
%D2%	YYMMDDHH
%D3%	YYMMDDHHMM
%L%	The backup level
======= ======================================

Configuration directives:
=========================

Configuration directives should preserve the same name across scripts  when
possible. Common directives are (using regexp):

============== ======================================================
DESTDIR		Where to backup to
DATABASES	The databases to backup, for DB backups
PASSWORD	To hold a password
.*USER		To hold a username (examples: PGUSER, MYUSER etc..)
============== ======================================================
 
NOTE! The directive USER should never be used. I repeat NEVER USE THE USER
DIRECTIVE! Same thing applies for other well known directives that may be
set by sh/bash.

Common configuration variables:
===============================

Common configuration variables are variables that are set in vbackup.conf
or in other configuration files. The vbackup.conf entry is the main one
but it may be overridden by individual configuration files.

=============== ============================================================
COMPRESS	Whether to compress the backup or not (yes/no, on/off, 1/0)
=============== ============================================================

Sample files:
=============

Sample configuration files are parsed by vbackup when "--rc --add" is used.
The configuration files are parsed using the following logic:

 * Ignore all comments lines from the top of the file until an empty line is
   found
 * If in the above block of comments there's a line '## No autoconfig' then
   the '--rc --add' will just copy the file and not try to autoconfigure it.
   This is useful for example in the case of the exec plugin.
 * All files of the sample configuration file end up in the target
   configuration file. This means that all comments and empty lines will be
   copyied over there.
 * Look for blocks of directives. All blocks are assumed to have some comments
   followed by one or more configuration directives.
 * All the comments of a block will be displayed to the user. Only lines that
   start with '# ' (hash and a space) are assumed to be comments.
 * When a line that doesn't start with '# ' is found then it's interpreted as
   a configuration directive.
 * If the line start with # (e.g: #DESTDIR="test") then it is assumed not to
   have a default value. However, a line of text will be displayed to the
   user saying: "Sample value: test".
 * If the line doesn't start with # (e.g: DESTDIR="test") then it is assumed
   that the value is the default value. It will be presented both as a sample
   line and as default value. If the user presses enter then he will use
   that value.

Wizard:
=======
Inside the wizard dir, each script may place a configuration file. This file
will describe a wizard step for initial configuration creation.

Each wizard configuration file includes::

 NAME="the name of the configuration script"
 PRI="Priority of configuration file"

w_do_ask()
  A function that will be called to ask for configuration parameters.
  This function may be called multiple times to answer different questions.
  The first argument mandates the question to me asked
  Initialy, this function is called with "ENABLE" as the first argument.
  All capital letter names should be reserved for vbackup usage. This function
  must set the variable ASK_NEXT to "" whenever the series of questions is
  finished, or to a name that will be passed as the first argument to the
  next function call.

  The last function call must return 0 to indicate that this method is
  active or 1 to indicate that it is not

  Example call tree:

   * do_ask ENABLE ==> Set ASK_NEXT=path
   * do_ask path ==> Set ASK_NEXT=level
   * do_ask level ==> Set ASK_NEXT="", return code=0
   * finished asking questions, method is enabled

  This function may also write to the temporary file $CFG::

   echo 'DIRS="/tmp"' >> $CFG
   echo '# Comment' >> $CFG

  This file will be available when w_get_config() is called.
  Typically, this should contain the majority of the configuration options
  except maybe from static options.

w_get_config()
  A function that should print to its stdout the configuration file.



