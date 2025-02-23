
INTRODUCTION
============

The `tout` script is a resource monitoring program for limiting time and
memory consumption of black-boxed processes under Linux.  It runs a command
you specify in the command line and watches for its memory and time
consumption, interrupting the process if it goes out of the limits, and
notifying the user with the preset message.

The killer feature of this script (and, actually, the reason why it appeared)
is that it not only watches the process spawned directly, but also keeps track
of its subsequently forked children.  You may choose if the scope of the
watched processes is constrained by process group or by the process tree.

`tout` may optionally detect hangups, or print time consumption breakdown.


## Note: CGroups

Linux now supports the CGroups feature, which is a much better method to track
memory and time usage and not limited by the issues listed below. This `tout`
script **does not use CGroups**. While this script continues to work, if
you are on Linux and need a more robust monitoring method you **should use something
based on CGroups**. For system services, have a look at [available systemd
options](https://www.freedesktop.org/software/systemd/man/systemd.resource-control.html).
For a simple command-line script to limit memory usage you could for example
use [runexec](https://github.com/sosy-lab/benchexec/blob/master/doc/runexec.md),
which is part of [BenchExec](https://github.com/sosy-lab/benchexec).


INSTALLATION
============

Installation is neither required nor supported.  Just place the script
wherever you feel convenient and invoke it as a usual Linux command.

You will need Perl 5 to run the script, and the /proc filesystem mounted.


USAGE
=====

Like most of such wrapping scripts (nice, ionice, nohup), invocation is:

  tout [options] command [arguments]

The basic options are:

* `-k T` - set up wallclock time limit to T seconds
* `-t T` - set up CPU+SYS time limit to T seconds
* `-m M` - set up virtual memory limit to M kilobytes
* `TIMEOUT_IDSTR` environment variable - a custom string prepended to the
  message about resource violation (to distinguish from the lines printed by
  the command itself).  The message itself may be:
  - `TIMEOUT` - time limit is exhausted
  - `MEM` - memory limit is exhausted
  - `HANGUP` - hangup detected (see below)
  - `SIGNAL` - the tout process was killed by a signal

 After the message the number of seconds the process has been running for is
 printed.


Advanced options:

* `-p .*regexp1.*,NAME1;.*regexp2.*,NAME2` - collect statistics for children
  with specified commands.  NAMEs define buckets, and regexps (Perl format)
  define matching children that fall into these buckets.

  If a pattern begins with `CHILD:`, then the runtime of the children of the
  matching process (match being performed with the rest of the pattern) is
  collected under this category.  Note that this is an only way to collect
  statistics of time consumed by the children that last for fractions of a
  second only.

* `-o outfile` - a file to dump bucket statistics collected by `-p` option.

* `--detect-hangups` - enable hangup detection.  If you have specified
  buckets through the `-p` option, then if the CPU time in any of the buckets
  does not increase during some time, the tout script reasons that the
  controlled process hanged up, and terminates it.

* `--no-info-on-success` - disable printing usage statistics if the
  controlled process has been successfully terminated.

* `--confess`, `-c` - when killing the controlled process, return its exit
  code or signal+128.  This also makes tout to wait until the controlled
  process is terminated.  Without this option, the script returns zero.

* `--memlimit-rss`, `-s` - monitor RSS (resident set size) memory limit

More options may be read in the script itself.  More documentation will be
added in the future releases!

Exit code of the script is the exit code of the controlled process.  If the
controlled process was killed by a signal, the exit code is 128+N, where N is
the number of the signal.  This simulates Bash exit code policy.  If the
controlled process was terminated by the tout script itself the script
returns zero because having the tout terminate the child is expected
behavior.  If you want the child's return code in such a situation (which may
be nonzero if the child handles SIGTERM), use `--confess` option.


EXAMPLES
========

Since you already have Perl to run the script itself, the examples will
utilize it.

Basic time limiting:

    ./tout -t 2 perl -e 'while ($i<100000000) {$i++;}'
    Outputs:
    TIMEOUT 2.04 CPU

Basic memory limiting (1000M of virtual memory):

    ./tout -m 1000000 perl -e 'while ($i<100000000) {$a->{$i} = $i++;}'
    Outputs:
    MEM 8.55

Limit both time and memory (adjust number to match the command above):

    ./tout -m 1000000 -t 9 perl -e 'while ($i<100000000) {$x->{$i} = $i++;}'
    Outputs:
    MEM 8.57
    ./tout -m 1000000 -t 8 perl -e 'while ($i<100000000) {$x->{$i} = $i++;}'
    Outputs:
    TIMEOUT 8.02 CPU

Limit time with a lot of short child processes:

    ./tout -t 2 perl -e 'while(1){ system qw(perl -e while($i<500){$i++;}); }'
    Outputs (in 4 seconds):
    TIMEOUT 2.01 CPU

Collect statistics for `heavy' processes:

    ./tout -p '.*perl.*,PERL' perl -e 'for (1..20_000_000) {$i++;}'
    Outputs:
    <time name="PERL">1400</time>

Collect statistics for `lightweight' children:

    ./tout -t 10 -p '.*perl.*,PERL;CHILD:.*perl.*,KIDS' perl -e 'for (1..2_000) {system qw(perl -e while($i<500000){$i++;}); $i++;}'
    Outputs:
    TIMEOUT 10.18 CPU
    <time name="PERL">640</time>
    <time name="KIDS">10160</time>

Lightweight children should be tracked with special `CHILD:` prefix in their
pattern, compare the above with:

    ./tout -t 10 -p '.*perl.*,PERL' perl -e 'for (1..2_000) {system qw(perl -e while($i<500000){$i++;}); $i++;}'
    Outputs:
    TIMEOUT 10.06 CPU
    <time name="PERL">830</time>

Why is the rest not shown in the bucket statistics? All processes spawned are
Perl-s, but the short-living ones aren't tracked fully, since tout doesn't
wake up often enough.

Detect hangups:

    ./tout --detect-hangups -p '.*sleep.*,SLEEP' -t 5 sleep 10000
    Outputs:
    HANGUP CPU 0.00 MEM 19760 MAXMEM 19760 STALE 6



IMPLEMENTATION DETAILS
======================

The script wakes up several times a second and checks if the process tree (or
group) controlled does not violate the limits.

More explanations how the script works and why it couldn't be implemented
differently may be found here:

http://coldattic.info/shvedsky/pro/blogs/a-foo-walks-into-a-bar/posts/40


KNOWN ISSUES
============

The script is slow.  It's implemented in Perl, but that's not the reason as
itself.  Performance of certain portions of it may easily be improved.  Our
measurements demonstrated that it consumes up to 2% of the CPU time during its
work.

The script can overlook some child processes (in case of a double fork, for example).
The resources used by these processes will not be measured and limits won't apply to them.
Furthermore, these child processes will not be killed when the script terminates.

Due to the use of sampling, all measurements are somewhat imprecise.
Sort-lived child processes and memory peaks can be overlooked.

Sometimes waitpid call from inside SIGALRM handler returns -1 for no apparent
reason.  This return value is ignored, and the appropriate warning is printed,
but the cause of such a behavior is still unknown.

SIGTERM-sleep-SIGKILL termination sequence is probably implemented poorly, and
it sometimes does not get to sending SIGKILL if SIGTERM doesn't kill the
process controlled.  The reasons are still unknown.

The interface is a mess, as the script had been under requirements-driven
development without a specific plan.


ACKNOWLEDGMENTS
===============

The script was initially developed in the Institute for System Programming of
Russian Academy of Sciences (http://ispras.ru/en/) for Linux Driver
Verification project (http://forge.ispras.ru/projects/ldv) in 2010-2011 by
Pavel Shved with some contributions from Alexander Strakh.



