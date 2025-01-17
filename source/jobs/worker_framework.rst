.. _worker framework:

worker framework
================

Purpose
-------

The Worker framework has been developed to meet two specific use cases:

-  many small jobs determined by parameter variations; the scheduler's
   task is easier when it does not have to deal with too many jobs.
-  job arrays: replace the ``-t`` for array requests; this was an
   experimental feature provided by the torque queue system, but it is
   not supported by Moab, the current scheduler.

Both use cases often have a common root: the user wants to run a program
with a large number of parameter settings, and the program does not
allow for aggregation, i.e., it has to be run once for each instance of
the parameter values. However, Worker's scope is wider: it can be used
for any scenario that can be reduced to a
`MapReduce <https://en.wikipedia.org/wiki/MapReduce>`__
approach.

This how-to shows you how to use the Worker framework.

Prerequisites
-------------

A (sequential) job you have to run many times for various parameter
values. We will use a non-existent program cfd-test by way of running
example.

Step by step
------------

We will consider the following use cases already mentioned above:

-  :ref:`parameter variations <parameter variations>`, i.e., many
   small jobs determined by a specific parameter set;
-  :ref:`job arrays <job arrays>`, i.e., each individual job got a
   unique numeric identifier.

.. _parameter variations:

Parameter variations
~~~~~~~~~~~~~~~~~~~~

Suppose the program the user wishes to run is 'cfd-test' (this program
does not exist, it is just an example) that takes three parameters, a
temperature, a pressure and a volume. A typical call of the program
looks like:

::

   cfd-test -t 20 -p 1.05 -v 4.3

The program will write its results to standard output. A PBS script (say
run.pbs) that would run this as a job would then look like:

::

   #!/bin/bash -l
   #PBS -l nodes=1:ppn=1
   #PBS -l walltime=00:15:00
   cd $PBS_O_WORKDIR
   cfd-test -t 20  -p 1.05  -v 4.3

When submitting this job, the calculation is performed or this
particular instance of the parameters, i.e., temperature = 20, pressure
= 1.05, and volume = 4.3. To submit the job, the user would use:

::

   $ qsub run.pbs

However, the user wants to run this program for many parameter
instances, e.g., he wants to run the program on 100 instances of
temperature, pressure and volume. To this end, the PBS file can be
modified as follows:

::

   #!/bin/bash -l
   #PBS -l nodes=1:ppn=8
   #PBS -l walltime=04:00:00
   cd $PBS_O_WORKDIR
   cfd-test -t $temperature  -p $pressure  -v $volume

Note that

#. the parameter values 20, 1.05, 4.3 have been replaced by variables
   $temperature, $pressure and $volume respectively;
#. the *number of processors per node* has been increased to 8 (i.e.,
   ppn=1 is replaced by ppn=8); and
#. the *walltime* has been increased to 4 hours (i.e., walltime=00:15:00
   is replaced by walltime=04:00:00).

The walltime is calculated as follows: one calculation takes 15 minutes,
so 100 calculations take 1,500 minutes on one CPU. However, this job
will use 7 CPUs (1 is reserved for delegating work), so the 100
calculations will be done in 1,500/7 = 215 minutes, i.e., 4 hours to be
on the safe side. Note that starting from version 1.3, a dedicated core
is no longer required for delegating work when using the ``-master``
flag. This is however not the default behavior since it is implemented
using features that are not standard. This implies that in the previous
example, the 100 calculations would be completed in 1,500/8 = 188
minutes.

The 100 parameter instances can be stored in a comma separated value
file (CSV) that can be generated using a spreadsheet program such as
Microsoft Excel, or just by hand using any text editor (do **not** use a
word processor such as Microsoft Word). The first few lines of the file
data.txt would look like:

::

   temperature,pressure,volume
   20,1.05,4.3
   21,1.05,4.3
   20,1.15,4.3
   21,1.25,4.3
   ...

It has to contain the names of the variables on the first line, followed
by 100 parameter instances in the current example. Items on a line are
separated by commas.

The job can now be submitted as follows:

::

   $ module load worker/1.5.0-intel-2014a
   $ wsub -batch run.pbs -data data.txt

Note that the PBS file is the value of the -batch option . The cfd-test
program will now be run for all 100 parameter instances—7
concurrently—until all computations are done. A computation for such a
parameter instance is called a work item in Worker parlance.

.. _job arrays:

Job arrays
~~~~~~~~~~

Worker also supports job array-like usage pattern since it offers a
convenient workflow.

A typical PBS script run.pbs for use with job arrays would look like
this:

::

   #!/bin/bash -l
   #PBS -l nodes=1:ppn=1
   #PBS -l walltime=00:15:00
   cd $PBS_O_WORKDIR
   INPUT_FILE=\"input_${PBS_ARRAYID}.dat\"
   OUTPUT_FILE=\"output_${PBS_ARRAYID}.dat\"
   word-count -input ${INPUT_FILE}  -output ${OUTPUT_FILE}

As in the previous section, the word-count program does not exist. Input
for this fictitious program is stored in files with names such as
input_1.dat, input_2.dat, ..., input_100.dat that the user produced by
whatever means, and the corresponding output computed by word-count is
written to output_1.dat, output_2.dat, ..., output_100.dat. (Here we
assume that the non-existent word-count program takes options -input and
-output.)

The job would be submitted using:

::

   $ qsub -t 1-100 run.pbs

The effect was that rather than 1 job, the user would actually submit
100 jobs to the queue system (since this puts quite a burden on the
scheduler, this is precisely why the scheduler doesn't support job
arrays).

Using worker, a feature akin to job arrays can be used with minimal
modifications to the PBS script:

::

   #!/bin/bash -l
   #PBS -l nodes=1:ppn=8
   #PBS -l walltime=04:00:00
   cd $PBS_O_WORKDIR
   INPUT_FILE=\"input_${PBS_ARRAYID}.dat\"
   OUTPUT_FILE=\"output_${PBS_ARRAYID}.dat\"
   word-count -input ${INPUT_FILE}  -output ${OUTPUT_FILE}

Note that

#. the *number of CPUs* is increased to 8 (ppn=1 is replaced by ppn=8);
   and
#. the *walltime* has been modified (walltime=00:15:00 is replaced by
   walltime=04:00:00).

The walltime is calculated as follows: one calculation takes 15 minutes,
so 100 calculation take 1,500 minutes on one CPU. However, this job will
use 7 CPUs (1 is reserved for delegating work), so the 100 calculations
will be done in 1,500/7 = 215 minutes, i.e., 4 hours to be on the safe
side. Note that starting from version 1.3 when using the ``-master``
flag, a dedicated core for delegating work is no longer required. This
is however not the default behavior since it is implemented using
features that are not standard. So in the previous example, the 100
calculations would be done in 1,500/8 = 188 minutes.

The job is now submitted as follows:

::

   $ module load worker/1.5.0-intel-2014a
   $ wsub -t 1-100  -batch run.pbs

The word-count program will now be run for all 100 input files—7
concurrently—until all computations are done. Again, a computation for
an individual input file, or, equivalently, an array id, is called a
work item in Worker speak. Note that in constrast to torque job arrays,
a worker job array submits a single job.

MapReduce: prologues and epilogue
---------------------------------

Often, an embarrassingly parallel computation can be abstracted to three
simple steps:

#. a preparation phase in which the data is split up into smaller, more
   manageable chuncks;
#. on these chuncks, the same algorithm is applied independently (these
   are the work items); and
#. the results of the computations on those chuncks are aggregated into,
   e.g., a statistical description of some sort.

The Worker framework directly supports this scenario by using a prologue
and an epilogue. The former is executed just once before work is started
on the work items, the latter is executed just once after the work on
all work items has finished. Technically, the prologue and epilogue are
executed by the master, i.e., the process that is responsible for
dispatching work and logging progress.

Suppose that 'split-data.sh' is a script that prepares the data by
splitting it into 100 chuncks, and 'distr.sh' aggregates the data, then
one can submit a MapReduce style job as follows:

::

   $ wsub -prolog split-data.sh  -batch run.pbs  -epilog distr.sh -t 1-100

Note that the time taken for executing the prologue and the epilogue
should be added to the job's total walltime.

Some notes on using Worker efficiently
--------------------------------------

#. Worker is implemented using MPI, so it is not restricted to a single
   compute node, it scales well to many nodes. However, remember that
   jobs requesting a large number of nodes typically spend quite some
   time in the queue.
#. Worker will be effective when

   -  work items, i.e., individual computations, are neither too short,
      nor too long (i.e., from a few minutes to a few hours); and,
   -  when the number of work items is larger than the number of CPUs
      involved in the job (e.g., more than 30 for 8 CPUs).

Monitoring a worker job
-----------------------

Since a Worker job will typically run for several hours, it may be
reassuring to monitor its progress. Worker keeps a log of its activity
in the directory where the job was submitted. The log's name is derived
from the job's name and the job's ID, i.e., it has the form
<jobname>.log<jobid>. For the running example, this could be
'run.pbs.log445948', assuming the job's ID is 445948. To keep an eye on
the progress, one can use:

::

   $ tail -f run.pbs.log445948

Alternatively, a Worker command that summarizes a log file can be used:

::

   $ watch -n 60 wsummarize run.pbs.log445948

This will summarize the log file every 60 seconds.

Time limits for work items
--------------------------

Sometimes, the execution of a work item takes long than expected, or
worse, some work items get stuck in an infinite loop. This situation is
unfortunate, since it implies that work items that could successfully
are not even started. Again, a simple and yet versatile solution is
offered by the Worker framework. If we want to limit the execution of
each work item to at most 20 minutes, this can be accomplished by
modifying the script of the running example.

::

   #!/bin/bash -l
   #PBS -l nodes=1:ppn=8
   #PBS -l walltime=04:00:00
   module load timedrun/1.0.1
   cd $PBS_O_WORKDIR
   timedrun -t 00:20:00 cfd-test -t $temperature  -p $pressure  -v $volume

Note that it is trivial to set individual time constraints for work
items by introducing a parameter, and including the values of the latter
in the CSV file, along with those for the temperature, pressure and
volume.

Also note that 'timedrun' is in fact offered in a module of its own, so
it can be used outside the Worker framework as well.

Resuming a Worker job
---------------------

Unfortunately, it is not always easy to estimate the walltime for a job,
and consequently, sometimes the latter is underestimated. When using the
Worker framework, this implies that not all work items will have been
processed. Worker makes it very easy to resume such a job without having
to figure out which work items did complete successfully, and which
remain to be computed. Suppose the job that did not complete all its
work items had ID '445948'.

::

   $ wresume -jobid 445948

This will submit a new job that will start to work on the work items
that were not done yet. Note that it is possible to change almost all
job parameters when resuming, specifically the requested resources such
as the number of cores and the walltime.

::

   $ wresume -l walltime=1:30:00 -jobid 445948

Work items may fail to complete successfully for a variety of reasons,
e.g., a data file that is missing, a (minor) programming error, etc.
Upon resuming a job, the work items that failed are considered to be
done, so resuming a job will only execute work items that did not
terminate either successfully, or reporting a failure. It is also
possible to retry work items that failed (preferably after the glitch
why they failed was fixed).

::

   $ wresume -jobid 445948 -retry

By default, a job's prologue is not executed when it is resumed, while
its epilogue is. 'wresume' has options to modify this default behavior.

Aggregating result data
-----------------------

In some settings, each work item produces a file as output, but the
final result should be an aggregation of those files. Although this is
not necessarily hard, it is tedious, but Worker can help you achieve
this easily since, typically, the file name produced by a work item is
based on the parameters of that work item.

Consider the following data file ``data.csv``:

::

   a,   b
   1.3, 5.7
   2.7, 1.4
   3.4, 2.1
   4.1, 3.8

Processing it would produce 4 files, i.e., ``output-1.3-5.7.txt``,
``output-2.7-1.4.txt``, ``output-3.4-2.1.txt``, ``output-4.1-3.8.txt``.
To obtain the final data, these files should be concatenated into a
single file ``output.txt``. This can be done easily using ``wcat``:

::

   $ wcat  -data data.csv  -pattern output-[%a%]-[%b%].txt  -output output.txt

The pattern describes the file names as generated by each work item in
terms of the parameter names and values defined in the data file
``data.csv``.

``wcat`` optionally skips headers of all of the first file when the
``-skip_first n`` option is used (*n* is the number of lines to skip).
By default, blank lines are omitted, but by using the ``-keep_blank``
options, they will be written to the output file. Help is available
using the ``-help`` flag.

Multithreaded work items
------------------------

When a cluster is configured to use CPU sets, using Worker to execute
multithreaded work items doesn't work by default. Suppose a node has 20
cores, and each work item runs most efficiently on 4 cores, then one
would expect that the following resource specification would work:

::

   $ wsub  -l nodes=10:ppn=5 -W x=nmatchpolicy=exactnode  -batch run.pbs  \\
           -data my_data.csv

This would run 5 work items per node, so that each work item would have
4 cores at its disposal. However, this will not work when CPU sets are
active since the four work item threads would all run on a single core,
which is detrimental for application performance, and leaves 15 out of
the 20 cores idle. Simply adding the -threaded option will ensure that
the behavior and performance is as expected:

::

    $ wsub -l nodes=10:ppn=5 -batch run.pbs -data my_data.csv -threaded 4

Note however that using multihreaded work items may actually be less
efficient than single threaded execution in this setting of many work
items since the thread management overhead will be accumulated.

Also note that this feature is new since Worker version 1.5.x.

Further information
-------------------

For the information about the most recent version and new features
please check the official `worker documentation`_ webpage.

For information on how to MPI programs as work items, please contact
your friendly system administrator.

This how-to introduces only Worker's basic features. The ``wsub``
command and all other Worker commands have some usage information that
is printed when the ``-help`` option is specified:

::

   ### error: batch file template should be specified
   ### usage: wsub  -batch <batch-file>          \\
   #                [-data <data-files>]         \\
   #                [-prolog <prolog-file>]      \\
   #                [-epilog <epilog-file>]      \\
   #                [-log <log-file>]            \\
   #                [-mpiverbose]                \\
   #                [-master]                    \\
   #                [-threaded]                  \\
   #                [-dryrun] [-verbose]         \\
   #                [-quiet] [-help]             \\
   #                [-t <array-req>]             \\
   #                [<pbs-qsub-options>]
   #
   #   -batch <batch-file>   : batch file template, containing variables to be
   #                           replaced with data from the data file(s) or the
   #                           PBS array request option
   #   -data <data-files>    : comma-separated list of data files (default CSV
   #                           files) used to provide the data for the work
   #                           items
   #   -prolog <prolog-file> : prolog script to be executed before any of the
   #                           work items are executed
   #   -epilog <epilog-file> : epilog script to be executed after all the work
   #                           items are executed
   #   -mpiverbose           : pass verbose flag to the underlying MPI program
   #   -verbose              : feedback information is written to standard error
   #   -dryrun               : run without actually submitting the job, useful
   #   -quiet                : don't show information
   #   -help                 : print this help message
   #   -master               : start an extra master process, i.e.,
   #                           the number of slaves will be nodes*ppn
   #   -threaded             : indicates that work items are multithreaded,
   #                           ensures that CPU sets will have all cores,
   #                           regardless of ppn, hence each work item will
   #                           have <total node cores>/ppn cores for its
   #                           threads
   #   -t <array-req>        : qsub's PBS array request options, e.g., 1-10
   #   <pbs-qsub-options>    : options passed on to the queue submission
   #                           command

Troubleshooting
---------------

The most common problem with the Worker framework is that it doesn't
seem to work at all, showing messages in the error file about module
failing to work. The cause is trivial, and easy to remedy.

Like any PBS script, a worker PBS file *has to be* in UNIX format!

If you edited a PBS script on your desktop, or something went wrong
during sftp/scp, the PBS file may end up in DOS/Windows format, i.e., it
has the wrong line endings. The PBS/torque queue system can not deal
with that, so you will have to convert the file, e.g., for file
'run.pbs'

::

   $ dos2unix run.pbs

.. include:: links.rst
