# logthat

**What?**

A program that remembers what you did to your data so you don't have to.

**Why?**

Remembering exactly what you did to your data is hard (especially after it's been a while). Computers are good at remembering things. It seems like a decent idea to have your computer keep track of what you did to your data.

**How?**

A C daemon that uses the inotify API as well as a command line program.

**Isn't this just `history | grep "data.csv"`?** 

Sort of. But by default bash only keeps the last 1000 commands you entered, which makes it hard to keep track of what you did months and thousands of commands ago.

**Couldn't this program be avoided by writing scripts for all data analysis?**

In a perfect world, yes. But the reality is it's often easier to write an awk one-liner and forget about it than it is to save your one-liner in a script.

**What sorts of things can we log?**

UNIX one-liners (long piped commands, awk commands, etc.), field specific commands (i.e. for bioinformatics: samtools/bcftools/whatever long commands)

**This already exists!!!**

Oh thank goodness. Could you send it to me?

**What's a daemon?**

A program that runs in the background.

---

Here's how I'm thinking this program will be used:

    $ cd data/
    $ lt init mydata.csv

`logthat` will now be watching programs and commandlines that access the file mydata.csv. Whenever a program/process accesses that file, it will log it in a hidden file (TBD: one log per directory (`data/.lt` or one log per file? `data/.mydata.csv.lt`). The log will be a JSON document with the following variables: `filename`, `path`, `command`, `time`.

You can view the log like so:

    $ lt view mydata.csv

This will pretty print all files access events (maybe through less?).

Comments? Tweet at me (@arundurvasula) or submit an issue.

---

# Implementation

`logthat` will consist of two main parts: 1) the daemon that watches files, and 2) the command line program that you interact with. 

1) the daemon

The daemon will implement inotify and watch the files (or directories!) listed in a file in the home directory `~/.logthat`. Whenever it sees a change in a file there, it will write to a log in that file's directory (`.lt`). 

For example, if `.logthat` contains `/home/arun/data/mydata.csv`, the daemon will write to a log file called `/home/arun/data/.lt`. Each directory will get its own `.lt` file. (Potential problem: what happens when you move files?)

That takes care of where to write, now we need to know what to write. The idea is to save commands that access a file. To do this, we use a combination of `lsof`, PIDs, and linux filesystems. So given that we have a path to a file, how do we get the command line used to access it? First we find which processes are using the file. `lsof -t <file>` will list the processes accessing a file. The commandline for this is stored in `/proc/$PID/cmdline`. So combining those, we have `cat /proc/$(lsof -t <file>)/cmdline`. 

A trivial example is given here: in one shell run 

    $ touch a.txt
    $ tail -f a.txt
    
Then open another shell and run

    $ cat /proc/$(lsof -t ./a.txt)/cmdline | tr "\0" " " 
    
The output should be `tail -f a.txt` (without a newline), which is what we want to log. Not sure why `\0` is used instead of spaces.

So given this, how do we create something that watches any arbitrary list of files?

1. Create a daemon that is always running. Set up an inotify watch on all files listed in `~/.logthat`.
2. If you see an interesting event (file opened, file modified, file deleted?), get the command line by `$ cat /proc/$(lsof -t ./a.txt)/cmdline | tr "\0" " "`
3. Log that command line into the appropriate directory log along with other interesting info (time, user?, etc.)

2) the command line program

This command line program will have a few commands.
`lt add <arg>` (or `lt init`, not sure which is better but need to pick only one) - adds the file or directory specified to `~/.logthat`. If nothing is specified, add the current directory.

`lt rm <arg>` - removes the file or directory specified in `~/.logthat`. If nothing is specified, remove the current directory. Note that this isn't like `git rm` and will not remove any files nor will it delete any logs.

`lt view <arg>` - view the file or directory supplied in pretty printed JSON. Should often be combined with `| less` to make output scrollable. If nothing is supplied it tries to read the `.lt` file in the current directory or outputs an error if there is none.

`lt restart` - restarts the daemon for whatever reason you may need (it got killed, it's acting weird, whatever).

---
# Development (meta)log

March 16th 2016

Whoops. There's a timing issue. The program has to access the file long enough to run the lsof command and get the output which doesn't happen on short commands. Also it doesn't seem like it captures the entire command when I run `lsof` in a process that I open.

Trivial example: https://gist.github.com/arundurvasula/53f6d430152a175e348e
Test like the following

    $ gcc inotifyt.c
    $ ./a.out

Open a new shell. `a.out` will watch the `/home/adurvasu/a.txt` for file opening. To get an event long enough to detect, run the following:

    $ awk '{while(1) print}' a.txt > /dev/null
    
(^+C to exit). `a.out` should print `Command: awk`. There are two issues here:

1. It doesn't print the whole command. I believe this is because all the spaces are represented as `\0`, which is the null character and signifies the end of a string in C. I need to figure out how to swap these out with real spaces.
2. The commands have to be run long enough for `a.out` to catch them. I don't know what the lower limit is, but it would be nice to catch all file openings, not just long ones. Though this shouldn't be a problem for long commands (i.e. an awk script though a VCF or a bcftools call). Also not a problem for less apparently.
