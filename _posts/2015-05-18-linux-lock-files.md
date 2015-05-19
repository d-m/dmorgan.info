---
layout: post
title: "Linux Lock Files"
modified: 2015-05-19 10:22:18 -0400
category: posts
tags: [linux, lock, lock file, flock, scripting, shell, bash]
image:
  feature: 
  credit: 
  creditlink: 
comments: 
share: 
---

Often times, running processes on a Linux system need to coordinate their operations to prevent conflicts or race conditions. For example, two processes may need to safely access a shared resource or a cron job may need to verify that a previous instance has completed before running again. Processes generally use the concept of file locking to serialize their actions. Cooperating processes acquire a lock on a lock file to indicate that it is their turn to run or manipulate a shared resource. The contents of the lock file is irrelevant; they exist purely to indicate that a resource is occupied.

Linux systems offer two kinds of locks: exclusive, or write, locks and shared, or read, locks. These locks behave differently:

 - If a process holds an exclusive lock on a file, then no other process can acquire a lock, shared or exclusive, on that file.
 - If a process holds a shared lock on a file, then no other process can acquire an exclusive lock on that file.
 - Multiple processes can hold a shared lock on a file simultaneously.

These locks are advisory on Linux systems, meaning that all processes must agree to respect locks held by a process. There are no safeguards to prevent an uncooperative process from ignoring a lock.

## A (forced?) analogy

A lock file can be thought of as a record player. While someone is putting on a record (obtaining an exclusive lock):

 - No one else in the room can listen to the music (obtain a shared lock).
 - No one else can pick out their own record to play (obtain an exclusive lock).

 Similarly, when everyone is listening to the music:

 - Anyone else can listen to the music (obtain a shared lock), too.
 - Anyone wanting to put on a different record (obtain an exclusive lock) must wait for everyone else to finish listening, first.

## Locking files with `flock`

One common way to lock a file on a Linux system is `flock`. The `flock` command can be used from the command line or within a shell script to obtain a lock on a file and will create the lock file if it doesn't already exist, assuming the user has the appropriate permissions. Some possible flags for `flock` are below:

 - *-x* Acquire an exclusive lock on a file (the default)
 - *-s* Acquire a shared lock on a file
 - *-n* Fail by exiting with an exit code of 1 instead of waiting if a lock on a file is not available
 - *-w seconds* If a lock is not available, wait a specified number of seconds before exiting with an exit code of 1
 - *-u* Drop a lock. A lock is typically dropped when the lock file is closed, so this flag is generally only needed in special cases

### Using `flock` in a shell script

The `flock` command can be used in a shell script as follows:

{% highlight bash linenos %}
#!/bin/bash
# exclusive_lock.sh

(
  flock -xn 200
  trap 'rm /var/lock/lockfile' 0
  RETVAL=$?
  if [ $RETVAL -eq 1 ] ; then
    echo $RETVAL
    exit 1
  else
    echo "sleeping"
    sleep 10
  fi
) 200>/var/lock/lockfile
{% endhighlight %}

{% highlight bash linenos %}
#!/bin/bash
# shared_lock.sh

(
  flock -sn 200
  trap 'rm /var/lock/lockfile' 0
  RETVAL=$?
  if [ $RETVAL -eq 1 ] ; then
    echo $RETVAL
    exit 1
  else
    echo "sleeping"
    sleep 10
  fi
) 200>/var/lock/lockfile
{% endhighlight %}

These two scripts are nearly identical: the first one attempts to acquire an exclusive lock while the second one attempts to acquire a shared lock. When used this way, `flock` takes a [file descriptor](https://en.wikipedia.org/wiki/File_descriptor): 200 in this example. The script attempts to obtain the lock in a non-blocking way, meaning that the script will exit immediately with an exit code of 1 if the lock is not available. It then opens `/var/lock/lockfile` and assigns the file descriptor 200 to it. The `trap` statement isn't needed for acquiring the lock, but ensures that the lock file is removed upon successful completion of the code block.

These scripts also show the advisory nature of file locking on Linux systems. The script itself is responsible for checking the return value of the `flock` statement and handle it appropriately.

#### Example 1: Acquiring an exclusive lock

In three terminals, `run exclusive_lock.sh` in the first, `exclusive_lock.sh` again in the second, and `shared_lock.sh` in the third. The output should look similar to the following:

~~~
> ./exclusive_lock.sh
> sleeping
~~~

~~~
> ./exclusive_lock.sh
> 1
~~~

~~~
> ./shared_lock.sh
> 1
~~~

Here, the first run of `exclusive_lock.sh` acquired an exclusive lock on `/var/lock/lockfile` and executed its sleep statement. Since the lock was already taken, neither the second run of `exclusive_lock.sh` nor the run of `shared_lock.sh` were able to obtain the lock and exited with an exit code of 1.

#### Example 2: Acquiring a shared lock

In three terminals, run `shared_lock.sh` in the first, `shared_lock.sh` in the second, and `exclusive_lock.sh` in the third. The output should look like the following:

~~~
> ./shared_lock.sh
> sleeping
~~~

~~~
> ./shared_lock.sh
> sleeping
~~~

~~~
> ./exclusive_lock.sh
> 1
~~~

The first run of `shared_lock.sh` acquired a shared lock on `/var/lock/lockfile`. Since multiple shared locks can exist simultaneously, the second run of `shared_lock.sh` also obtained a lock. However, since an exclusive lock cannot be obtained while a shared lock exists, the run of `exclusive_lock.sh` exited with an exit code of 1.
