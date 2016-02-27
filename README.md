# logthat

**What?**

A program that remembers what you did to your data so you don't have to.

**Why?**

Remembering exactly what you did to your data is hard (especially after it's been more than a month). Computers are good at remembering things. It seems like a decent idea to have your computer keep track of what you did to your data.

**How?**

TBA. INotify API? Watch `history`? I suspect this will have to be written in C.

**Isn't this just `history | grep "data.csv"`?** 

Sort of. But by default bash only keeps the last 1000 commands you entered, which makes it hard to keep track of what you did months and thousands of commands ago.

**Couldn't this program be avoided by writing scripts for all data analysis?**

In a perfect world, yes. But the reality is it's easier to write an awk one-liner and forget about it than it is to save your one-liner in a script.

**What sorts of things can we log?**

UNIX one-liners (long piped commands, awk commands, etc.), field specific commands (i.e. for bioinformatics: samtools/bcftools/whatever long commands), scripts (Python/R/whatever).

**This already exists!!!**

Oh thank goodness. Could you send it to me?

---

Here's how I'm thinking this program will be used:

    $ cd data/
    $ lt init mydata.csv

`logthat` will now be watching programs and commandlines that access the file mydata.csv. Whenever a program/process accesses that file, it will log it in a hidden file (TBD: one log per directory (`data/.lt` or one log per file? `data/.mydata.csv.lt`). The log will be a JSON document with the following variables: `filename`, `path`, `command`, `time`.

You can view the log like so:

    $ lt view mydata.csv

This will pretty print all files access events (maybe through less?).

Comments? Tweet at me (@arundurvasula) or submit an issue.
