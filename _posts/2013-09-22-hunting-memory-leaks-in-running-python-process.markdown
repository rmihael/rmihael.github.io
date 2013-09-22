---
layout: post
title:  "Hunting memory leaks in running Python process"
description: "What it takes to connect memory profile to running Python process"
og_image_url: "/assets/img/leak-bali.jpg"
tags: python, debug
visible: true
---
<img src="{{ page.og_image_url }}" align="right" width="50%"/>

Bad things happens more often then we want. If you work on product running in production then it's just a matter of time when you'll see some process eating much more memory then it expected to. In a lucky case it's possible to track down a sequence of events that lead to a leak and reproduce it in a controlled environment with full power of debugging tools. However more often then not a leak is caused by something unknown that only happens in production and not on QA. That's exactly a kind of situation I found myself recently staring at 1.2 gigs Python process that normally consuming only 20-30 megs. People of Indonesia probably knew something when named one of their mythological monster [leak](http://en.wikipedia.org/wiki/Leyak).

Without any clues about what caused a leak, memory state of a process is the most obvious place to get some insight. It's possible to get full core dump of a process by sending SIGABRT with `kill -SIGABRT <pid>` or through [GDB](https://www.gnu.org/software/gdb/): `gdb --pid=<pid> -ex generate-core-file`. However Python code dumps are not easy to work with (at least for me). 

There's some special tools designed to help with memory leaks in Python. I'm aware of [Heapy][heapy] and [Meliae][meliae] and probably there's some more. But these tools are intended to be used from a scope of a process under investigation. That's fine if it's possible to modify a source code and reproduce a memory leak. But it's not helping a lot with an existing process in faulty state.

Shooting for the moon I would like to have a Python's shell in the context of a troublesome process with access to either Heapy or Meliae toolset. Let's see what can be done to get there.

## Attaching shell ##

One great tool to interact with running Python processes is [Pyrasite](http://pyrasite.readthedocs.org/). It's essentially the GDB wrapper that allows to run arbitrary code in the context of some Python process.

Installing Pyrasite doesn't take much. GDB is the only prerequisite. Version requirement on [GitHub page](https://github.com/lmacken/pyrasite) is bit unclear: "gdb (version 7.3+ (or RHEL5+))". In practice it's been working fine for me on CentOS 5+, despite bundled GDB version 7.2. Package itself is available through PyPi, so pip/easy_install do their job nicely.

After installing Pyrasite it's taking just one command to connect to the running process:

{% highlight bash %}
$ sudo pyrasite-shell 27817
Pyrasite Shell 2.0
Connected to '/usr/bin/python27 bin/messages_rest_api 5637 8'
Python 2.7.3 (default, May 18 2012, 22:11:42)
[GCC 4.4.6 20110731 (Red Hat 4.4.6-3)] on linux2
Type "help", "copyright", "credits" or "license" for more information.
(DistantInteractiveConsole)
>>>
{% endhighlight %}

`sudo` is required to get into someone else's process, but other then that it's just feels like magic. The following snippet can be used to make sure if it's really the process I'm looking for:

{% highlight python %}
import sys, traceback

for thread, frame in sys._current_frames().items():
    print('Thread 0x%x' % thread)
    traceback.print_stack(frame)
    print()
{% endhighlight %}

Typing it in will print all threads that process should have plus one dedicated to this new shell:

{% highlight python %}
Thread 0x7f5c341d8700
  File "/usr/lib64/python2.7/threading.py", line 524, in __bootstrap
    self.__bootstrap_inner()
  File "/usr/lib64/python2.7/threading.py", line 551, in __bootstrap_inner
    self.run()
  File "<string>", line 167, in run
  File "/usr/lib64/python2.7/code.py", line 243, in interact
    more = self.push(line)
  File "/usr/lib64/python2.7/code.py", line 265, in push
    more = self.runsource(source, self.filename)
  File "/usr/lib64/python2.7/code.py", line 87, in runsource
    self.runcode(code)
  File "/usr/lib64/python2.7/code.py", line 103, in runcode
    exec code in self.locals
  File "<console>", line 3, in <module>
()
{% endhighlight %}

Now about actual profiling...

## Attaching memory profiler ##

I prefer to have some debugging tools included to production deployments. Usually they include IPython, [IPDB](https://pypi.python.org/pypi/ipdb) and [Heapy][heapy]. If they were around before process started they will be available for `import` immediately. It's not always the case, unfortunately. Heapy is default weapon-of-choice for memory profiling, so let's make it available in pyrasite shell.

No surprise, Heapy can be installed with standard tools like easy_install/pip. It's part of [guppy](https://pypi.python.org/pypi/guppy) package. However even after installing, it still will not be available in pyrasite shell. It's just not in the packages search path. In order to make it available, the package directory must be added to process' `sys.path`:

{% highlight python %}
>>> from guppy import heapy
Traceback (most recent call last):
  File "<console>", line 1, in <module>
ImportError: No module named guppy

>>> import sys
>>> sys.path.append("/usr/lib/python2.7/site-packages/guppy-0.1.10-py2.7-linux-x86_64.egg")
>>> from guppy import hpy
>>> hp = hpy()
>>> hp.heap()
Partition of a set of 208391 objects. Total size = 27694176 bytes.
 Index  Count   %     Size   % Cumulative  % Kind (class / dict of class)
     0  97049  47  8741208  32   8741208  32 str
     1  52668  25  4466568  16  13207776  48 tuple
     2    777   0  1831128   7  15038904  54 dict of module
     3   1988   1  1796392   6  16835296  61 type
     4  13807   7  1767296   6  18602592  67 types.CodeType
     5  13660   7  1639200   6  20241792  73 function
     6   1531   1  1573384   6  21815176  79 dict (no owner)
     7   1988   1  1507040   5  23322216  84 dict of type
     8   2586   1   406928   1  23729144  86 list
     9    372   0   390624   1  24119768  87 dict of class
<533 more rows. Type e.g. '_.more' to view.>
{% endhighlight %}

Now that's looks good. From that point it's possible to use full power of Heapy to track down the root cause. Other tools can be added in similar way. I found that [Meliae][meliae] while somewhat painful to install helps me better in [complicated cases](https://github.com/celery/kombu/issues/159). There's [good tutorial](http://jam-bazaar.blogspot.com/2010/08/step-by-step-meliae.html) about this tool.

[meliae]: https://launchpad.net/meliae
[heapy]: http://guppy-pe.sourceforge.net/#Heapy