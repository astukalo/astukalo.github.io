---
layout: post
title: Linux magic. How to redirect stdout and stderr to file and add timestamp
tags: [shell, linux, scripting]
comments: true
---

In one of our java projects we have a 3rd party lib that sometimes logs to stderr and stdout.
Unfortunately, these log entries are quite important to ignore and
of course we would like to know **when** shit happens.

So, I have written this shell script to redirect output to file and to add timestamps there<!--more-->:

{% gist 080964d1553dca2349e9d98b10f933a6 %}

At line #2 of the script we get the current directory.
Then we set JAVA_HOME variable and define some java properties to print GC details.
This is obvious.

The topic itself is at lines #6-7.

```sh
{ { ${JAVA_HOME}/bin/java ${java_properties} -jar ${DIR}/lib/app.jar 2>&1 & echo $! > ${DIR}/app.pid; } |\
while read line; do echo "$(date +"%Y-%m-%d %T") $line" >> ${DIR}/logs/stdout.log; done; } &
```

Let's go step by step and try to understand the magic inside.

So, here we execute our jar and redirect stderr to stdout and run it in background

```sh
${JAVA_HOME}/bin/java ${java_properties} -jar ${DIR}/lib/app.jar 2>&1 &
```

Furthermore,

```sh
${JAVA_HOME}/bin/java ${java_properties} -jar ${DIR}/lib/app.jar 2>&1 & echo $! > ${DIR}/app.pid;
```
adding echo, dollar sign, exclamation we get the process id (PID) and print it to app.pid.
That is just to be able to stop the process like this:

```sh
kill $(cat ${DIR}/app.pid)
```

The next step is to add timestamps to each log entry. We can achieve this with

``` sh
while read line; do echo "$(date +"%Y-%m-%d %T") $line" >> ${DIR}/logs/stdout.log; done;
```
Here we use piping to redirect an output of the previous command to `while` loop.
In this loop we echo current time formatted (`date +"%Y-%m-%d %T"`) and append the result to log file with `>>`.

The next trick that we have to do is to wrap everything we did into curly braces `{  }` to execute it in the current shell context
and to run it in background (`... &`).

Voila, this script will print us log entries with timestamp.

