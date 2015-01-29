## Why doesn't R print in red on some systems?

I have R installed on:

* my laptop
* desktop1 (Fedora release 21 (Twenty One))
* desktop2 (CentOS release 6.6 (Final))

I have been switching between them when doing coursework.  It's quickly begun to drive me (minorly) nuts that manual pages using quotes don't work properly on the latter two systems, nor does the `swirl()` lesson text show up on red.
I know this isn't wholly rational, but the context of proper formatting helps on the laptop, so I want it to help on the desktop.

_Environment_
* laptop
  * iTerm
* desktops: connected to via ssh, and usually screen
  * iTerm from home, PuTTY from the office
  * desktop1:
    * TERM=xterm
    * LANG=en_GB.UTF-8
    * ls --color: shows colors
    * man man: quotes, bold, underline all work
    * R message(): not in red
    * R help q: quotes garbled (until I fixed PuTTY, then fine)
  * desktop2:
    * TERM=xterm
    * LANG=en_GB.ISO_8859-1
    * ls --color: shows colors
    * man man: quotes, bold, underline all work
    * R message(): not in red
    * `Rscript -e "help(q)"` -- quotes fine (with fixed or un-fixed PuTTY)

At some point I figured out that the LANG setting was causing quotes to be garbled.  These settings seem to fix it:
* PuTTY: Window -> Translation -> Character set change from "ISO-8859-1:1998 (Latin-1, West Europe)" to "UTF-8"
* LANG=en_US.UTF-8

However, I still haven't figured out the coloring. I tried a number of TERM settings, but nothing took.
```
foreach t (xterm xterm-color vt100 vt220); do export TERM=$t; echo $TERM; Rscript -e "message('hi')"; done
```

I know it pops up when you first start R on the command line, but more definitely when you start `swirl()` -- so I dug into the [swirl() source code](https://github.com/swirldev/swirl) and found that [swirl's printing method, `swirl_out()`](https://github.com/swirldev/swirl/blob/master/R/utilities.R) uses `message()`.  I realized then that `message()` output is in red!

So then I used `debug(message)` and `message("hey")` to find this key line:
```
cat(conditionMessage(c), file = stderr(), sep = "")
```

This leads me to the conclusion that R is coloring `stderr()` red.  But I don't yet know how.  Maybe the [R source](https://github.com/wch/r-source) will help?

```
https://github.com/wch/r-source/blob/07ac1e21db175ac877530a5d0105906911e56c18/src/gnuwin32/system.c
says
R_Consolefile = stderr; /* used for errors */

https://github.com/wch/r-source/blob/663ddf40c6eec16067bd33170dd8f7d811d03aad/src/main/printutils.c
says:
 * All printing in R is done via the functions Rprintf and REprintf
 * or their (v) versions Rvprintf and REvprintf.
 * These routines work exactly like (v)printf(3).  Rprintf writes to
 * ``standard output''.	 It is redirected by the sink() function,
 * and is suitable for ordinary output.	 REprintf writes to
 * ``standard error'' and is useful for error messages and warnings.
 * It is not redirected by sink().
 *
 *  See ./format.c  for the  format_FOO_  functions which provide
 *	~~~~~~~~~~  the	 length, width, etc.. that are used here.
 *  See ./print.c  for do_printdefault, do_prmatrix, etc.
also has:
R_WriteConsole(p, (int) strlen(p));

not helpful:
https://github.com/wch/r-source/blob/master/src/main/print.c

looks like the stderr output routines but nothing about escape codes for red:
https://github.com/wch/r-source/blob/master/src/main/connections.c

Maybe it's a behavior of iTerm?
```
