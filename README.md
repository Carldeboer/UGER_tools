# UGER_tools
Tools for efficiently using UGER


This tool set is comprised of two tools:

`submitlog` is a wrapper for qsub that takes care of most of the nitty gritty details of using qsub and simultaneously logs the command you ran, the date, the qsub command, and the job ID in a file.

`newGEArrayJob` simply sets up a new batch file to run an array job.

`csj` watches the status of the jobs submitted from the current directory, refreshing every 2 seconds (using qstat)

# Installation
Simply `git clone <thisURL>` to download
and then you can create a shortcut (link) to your bin: `ln -s <pathToEachScriptFile> <pathToYourBin>`

You can also either copy or move them to your bin.  Only the files `newGEArrayJob`, `newGEHead`, `csj`, `checkSubbedJobs`, and `submitlog` are needed.

Your bin is `~/bin/.`. 

Submitlog uses python3 and you may need to install some of the scripts dependencies, including [argparse](https://pypi.python.org/pypi/argparse).  On the cluster, you must install all packages locally (i.e. using the `python3 setup.py install `**`--user`**).  You then would also need to include this install location in PYTHONPATH (i.e. put `export PYTHONPATH=$HOME/.local/lib` in your `~/.my.bashrc` file and re-login.

On the Broad cluster, you can simply type `use Python-3.4` to enable python3 and run submitlog. If you also use python < 3 (e.g. 2.7), you can instead `use Anaconda` and, at the time of writing, `python` will then refer to python 2.7, while `python3` will refer to python 3.5.

## Dedicated resources


If you have dedicated resources that you can access and you want to make using them default, you should customize submitlog to use it by modifying the line:
```parser.add_argument('-P',dest='project', metavar='<project>',help='project [default=""]', required=False, default = "");```

to 

```parser.add_argument('-P',dest='project', metavar='<project>',help='project [default="myDedicatedQueue"]', required=False, default = "myDedicatedQueue");```

where `myDedicatedQueue` is the name of the dedicated resources.


# `submitlog`
This is a general use script and comes with the following options:
```
$ submitlog -h
usage: submitlog [-h] [-q <queue>] [-e <errFile>] -o <outFile> [-m <memory>]
                 [-t <tasks>] [-op <otherParam>] [-P <project>]
                 [-p <priority>] [-N <name>] [-hosts <hosts>] [-n <threads>]
                 [-v]
                 <command> [<command> ...]

submit a job and log it.

positional arguments:
  <command>         The command to be run

optional arguments:
  -h, --help        show this help message and exit
  -q <queue>        run queue {long|short}
  -e <errFile>      Where to output stderr [default=stdout]
  -o <outFile>      Where to output stdout
  -m <memory>       RAM required (in GB)
  -t <tasks>        tasks for GE - either a file with one task per line or
                    1-<nTasks>
  -op <otherParam>  other qsub parameters, in quotes
  -P <project>      project [default="test"]
  -p <priority>     priority from -1023 to 0, with - being lower priority
                    (default 0)
  -N <name>         the job name
  -hosts <hosts>    server hosts, comma separated
  -n <threads>      CPU threads
  -v                Verbose output?
  ```
  
Some words of warning:
`-n <threads>`, the current grid engine uses something called parallel environments and I haven't quite figured out what the differences between them are, so this defaults to using only one.

`-P <project>` doesn't seem to matter any more, so should generally be unused.

`-o <outFile` should not be re-used for different jobs - what is actually run is a file `<outFile>.sh`, so if two different jobs are trying to run different commands with the same output file, the second one will overwrite the first in `<outFile>.sh` and so both jobs could potentially run the second job's commands.

`-hosts <hosts>` - haven't used this since LSF, don't count on it to work


## Examples
Here are a couple examples of submitlog.
```bash
$ submitlog -m 1 -q short -o test_echo.olog 'echo "this is how you use submitlog"'  # a job with 1GB RAM and using the short queue
Your job 907994 ("echo "this is how you use submitlog"") has been submitted

$ cat submitlog.log # see what was written to the log file
Fri Jan 15 13:56:24 2016
~/testDir
submitlog -m 1 -q short -o test_echo.olog 'echo "this is how you use submitlog"'
qsub  -b y -cwd  -q short -o test_echo.olog -l m_mem_free=1g -e test_echo.olog -N 'echo "this is how you use submitlog"' ./test_echo.olog.sh
Your job 907994 ("echo "this is how you use submitlog"") has been submitted

$ cat test_echo.olog.sh # this file is created and run behind the scenes and is what is actually run, which is why you should not run two simultaneous jobs with the same output file
#!/bin/bash -l
use Python-2.7; use .python-2.7.1-sqlite3-rtrees; use .zlib-1.2.6; use .hdf5-1.8.9; use .graphviz-2.28.0; use .db-4.7.25; use .tcltk8.5.9; use GCC-5.1; use .gcc-5.1.0; use reuse; use UGER; use default++; use .aliases++; use default; use .local; use .broad; use .hostname; use .lang; 
echo Job $JOB_ID started on $HOST: `date`
echo "this is how you use submitlog"
echo Job finished: `date`


$ cat test_echo.olog # See what is contained in the output file
/broad/software/dotkit/bash/use: line 35: unalias: ish: not found
Job 907994 started on hw-uger-1079: Fri Jan 15 13:56:35 EST 2016
this is how you use submitlog
Job finished: Fri Jan 15 13:56:35 EST 2016
```

# `newGEArrayJob`
This file simply simply copies the contents of newGEHead to a new file of your choosing (specified as the only parameter to the script), and opens it in vim for editing.

## Example
Here, I will show two common uses with trivial examples.

```bash
$ cat tasksNames.txt # I have a set of things I want to perform some task on
sample1
sample2
sample3
sample4

$ newGEArrayJob echoNameInFile.sh 
```
At this point, vim opens (you can change this by modifying the script).
Initially the script contains the following:

```bash
#!/bin/bash -l
#insert use commands here
#  $1 is the task file

# input the $SGE_TASK_IDth line of this file into $id
export id=`awk "NR==$SGE_TASK_ID" $1`
#split id on whitespace into array that can be accessed like ${splitID[0]}
#splitID=($id)
#export id=${splitID[0]}
echo  $SGE_TASK_ID : $id
#redirect stdout and stderr to file
export logDir=""
exec 1>>$logDir/$id.olog
exec 2>&1
set -e


if [ ! -e $logDir/$id.done ]
then
	echo Job $JOB_ID:$SGE_TASK_ID started on $HOST: `date`
	#PLACE COMMANDS HERE

	touch $logDir/$id.done
	echo Job finished: `date`
else
	echo Job already done. To redo, rm $logDir/$id.done
fi
```
I edit it to be the following:
```bash
#!/bin/bash -l
#insert use commands here
#  $1 is the task file

# input the $SGE_TASK_IDth line of this file into $id
export id=`awk "NR==$SGE_TASK_ID" $1`
#split id on whitespace into array that can be accessed like ${splitID[0]}
#splitID=($id)
#export id=${splitID[0]}
echo  $SGE_TASK_ID : $id
#redirect stdout and stderr to file
export logDir="example1" #output logs to current directory
mkdir -p $logDir # make the directory if it doesn't exist already
exec 1>>$logDir/$id.olog  ##this is where the output of this script will go
exec 2>&1
set -e


if [ ! -e $logDir/$id.done ] #this line means that the commands included below are only run if the file $logDir/$id.done does not exist so that you can re-run the command if some failed for some reason
then
	echo Job $JOB_ID:$SGE_TASK_ID started on $HOST: `date`
	echo $id >$id.example1.txt
	touch $logDir/$id.done
	echo Job finished: `date`
else
	echo Job already done. To redo, rm $logDir/$id.done
fi
```
Now I run it:
```bash
$ submitlog -q short -m 1 -o doExample1.olog -t tasksNames.txt ./echoNameInFile.sh tasksNames.txt 
Your job-array 908319.1-4:1 ("..echoNameInFile.sh tasksNames.txt") has been submitted

$ cat doExample1.olog 
/broad/software/dotkit/bash/use: line 35: unalias: ish: not found
/broad/software/dotkit/bash/use: line 35: unalias: ish: not found
/broad/software/dotkit/bash/use: line 35: unalias: ish: not found
/broad/software/dotkit/bash/use: line 35: unalias: ish: not found
4 : sample4
3 : sample3
1 : sample1
2 : sample2

$ ls example1/
sample1.done          sample1.olog  sample2.example1.txt  sample3.done          sample3.olog  sample4.example1.txt
sample1.example1.txt  sample2.done  sample2.olog          sample3.example1.txt  sample4.done  sample4.olog

$ cat example1/sample2.example1.txt # it echoed the sample $id into the file I specified
sample2
$ cat example1/sample2.olog # this is what the output looks like
Job 908367:2 started on hw-uger-1084: Fri Jan 15 14:18:47 EST 2016
Job finished: Fri Jan 15 14:18:47 EST 2016
```

So now a more complex example.  Say you have your different tasks, but you want to do slightly different things for each task, with a tab-delimited file containing what things you want to do:
```bash
$ cat tasksNamesOutsTimes.txt 
sample1	outFileA	6
sample2	outFileB	7
sample3	outFileC	8
sample4	outFileD	9
```
So this file tells me the sample name, the output file, and how many times the sample name should be printed in the output file.
My new arrayjob is as follows:
```

```

This little snippit takes the current line in the task list and splits it up into an array, taking the first element in the array as the job $id
```bash
splitID=($id)
export id=${splitID[0]}
```
Now, I run it:
```bash
$ submitlog -q short -m 1 -o doExample2.olog -t tasksNamesOutsTimes.txt ./echoNTimes.sh tasksNamesOutsTimes.txt 
Your job-array 908493.1-4:1 ("..echoNTimes.sh tasksNamesOutsTimes.txt") has been submitted

$ ls example2/  # check the output files
outFileA.example2.txt  outFileC.example2.txt  sample1.done  sample2.done  sample3.done  sample4.done
outFileB.example2.txt  outFileD.example2.txt  sample1.olog  sample2.olog  sample3.olog  sample4.olog

$ cat example2/outFileB.example2.txt # seven instances of the sample name!
sample2
sample2
sample2
sample2
sample2
sample2
sample2

$ submitlog -q short -m 1 -o doExample2.olog -t tasksNamesOutsTimes.txt ./echoNTimes.sh tasksNamesOutsTimes.txt ## rerun the script
Your job-array 908516.1-4:1 ("..echoNTimes.sh tasksNamesOutsTimes.txt") has been submitted

$ cat example2/sample2.olog # the first time I ran the script, it worked, so the second time it skipped execution
Job 908493:2 started on hw-uger-1083: Fri Jan 15 14:32:04 EST 2016
Job finished: Fri Jan 15 14:32:04 EST 2016
Job already done. To redo, rm example2/sample2.done
```

Finally, check out what was logged by submitlog:
```bash
$ cat submitlog.log 

Fri Jan 15 13:56:24 2016
~/testDir
submitlog -m 1 -q short -o test_echo.olog 'echo "this is how you use submitlog"'
qsub  -b y -cwd  -q short -o test_echo.olog -l m_mem_free=1g -e test_echo.olog -N 'echo "this is how you use submitlog"' ./test_echo.olog.sh
Your job 907994 ("echo "this is how you use submitlog"") has been submitted

Fri Jan 15 14:18:44 2016
~/testDir
submitlog -q short -m 1 -o doExample1.olog -t tasksNames.txt ./echoNameInFile.sh tasksNames.txt
qsub  -b y -cwd  -q short -o doExample1.olog -l m_mem_free=1g -e doExample1.olog -N '..echoNameInFile.sh tasksNames.txt' -t 1-4 './echoNameInFile.sh tasksNames.txt'
Your job-array 908367.1-4:1 ("..echoNameInFile.sh tasksNames.txt") has been submitted

Fri Jan 15 14:31:58 2016
~/testDir
submitlog -q short -m 1 -o doExample2.olog -t tasksNamesOutsTimes.txt ./echoNTimes.sh tasksNamesOutsTimes.txt
qsub  -b y -cwd  -q short -o doExample2.olog -l m_mem_free=1g -e doExample2.olog -N '..echoNTimes.sh tasksNamesOutsTimes.txt' -t 1-4 './echoNTimes.sh tasksNamesOutsTimes.txt'
Your job-array 908493.1-4:1 ("..echoNTimes.sh tasksNamesOutsTimes.txt") has been submitted

Fri Jan 15 14:34:25 2016
~/testDir
submitlog -q short -m 1 -o doExample2.olog -t tasksNamesOutsTimes.txt ./echoNTimes.sh tasksNamesOutsTimes.txt
qsub  -b y -cwd  -q short -o doExample2.olog -l m_mem_free=1g -e doExample2.olog -N '..echoNTimes.sh tasksNamesOutsTimes.txt' -t 1-4 './echoNTimes.sh tasksNamesOutsTimes.txt'
Your job-array 908516.1-4:1 ("..echoNTimes.sh tasksNamesOutsTimes.txt") has been submitted
```

#`csj` and `checkSubbedJobs`
`checkSubbedJobs` shows the current submitted jobs (running/queued) that were submitted from the current directory (it looks for them in ./submitlog.log). `csj` uses `checkSubbedJobs` to show the current running/queued jobs, refreshing every 2 seconds.  I almost never use `checkSubbedJobs` directly.
