% Tuning Rundeck

### File descriptors

The Rundeck server process opens a number of files during normal operation. These
include system and java libraries, logs, and sockets.
Your system restricts the number of open file handles per process
but these limitations can be adjusted.

If your installation attempts to exceed the limit, you will see an error
like the one shown below in your [service.log](logging.html) file.

    Too many open files


_On Linux nodes_

List the current limit with the [ulimit](https://ss64.com/bash/ulimit.html) command:

~~~~~ {.bash}
ulimit -n
~~~~~~

If the limit is low (eg ``1024``) it should be raised.

You can get the current number of open file descriptors used by the
Rundeck server process with [lsof](https://linux.die.net/man/8/lsof):

~~~~~ {.bash}
lsof -p <rundeck pid> | wc -l
~~~~~

Increase the limit for a wide margin.
Edit [/etc/security/limits.conf](https://ss64.com/bash/limits.conf.html) file
to raise the hard and soft limits. Here they are raised to ``65535`` for
the "rundeck" system account:

~~~~~ {.bash}
rundeck hard nofile 65535
rundeck soft nofile 65535
~~~~~


The system file descriptor limit is set in /proc/sys/fs/file-max.
The following command will increase the limit to 65535:

~~~~~ {.bash}
echo 65535 > /proc/sys/fs/file-max
~~~~~

In a new shell, run the ulimit command to set the new level:

~~~~~ {.bash}
ulimit -n 65535
~~~~~

The ulimit setting can be set in the [rundeckd](startup-and-shutdown.html#launcher)
startup script, or [profile](../configuration/configuration-file-reference.html#profile).

Restart Rundeck.

### Java heap size

The ``rundeckd`` startup script sets initial and maximum heap sizes
for the server process. For many installations it will be sufficient.

If the Rundeck JVM runs out of memory, the following error occurs:

    Exception in thread "main" java.lang.OutOfMemoryError: Java heap space

Heap size is governed by the following startup parameters:
``-Xms<initial heap size>`` and ``-Xmx<maximum heap size>``


You can increase these by updating the Rundeck [profile](../configuration/configuration-file-reference.html#profile).
To see the current values, grep the ``profile`` for
the Xmx and Xms patterns:

**Launcher installs:**

~~~~~ {.bash}
egrep '(Xmx|Xms)' $RDECK_BASE/etc/profile
~~~~~

**RPM installs:**

~~~~~ {.bash}
egrep '(Xmx|Xms)' /etc/rundeck/profile
~~~~~

The default settings initialized by the installer
sets these to 1024 megabytes maximum
and 256 megabytes initial:

~~~~~ {.bash}
export RDECK_JVM="$RDECK_JVM -Xmx1024m -Xms256m"
~~~~~

_Sizing advice_

Several factors drive memory usage in Rundeck:

* User sessions
* Concurrent threads
* Concurrent jobs
* Number of managed nodes

For example, if your installation has dozens of active users
that manage a large environment (1000+ nodes), and has
sufficient system memory, the following sizings might be more suitable:

~~~~~ {.bash}
export RDECK_JVM="$RDECK_JVM -Xmx4096m -Xms1024m"
~~~~~

### Quartz job threadCount

The maximum number of threads used by Rundeck for concurrent jobs
by default is set to ``10``.

You can change this value, by updating the
`rundeck-config.properties` file.

Please refer to the Quartz site for detailed information:
[Quartz - Configure ThreadPool Settings][1].

[1]:http://www.quartz-scheduler.org/documentation/quartz-2.x/configuration/ConfigThreadPool.html#configure-threadpool-settings

#### Update rundeck-config

Use the properties mentioned in the Quartz documentation, but **replace** the `org.quartz` prefix, with the prefix `quartz`.

e.g. in `rundeck-config.properties` :

~~~ {.properties}
quartz.threadPool.threadCount = 20
~~~

Set the threadCount value to the max number of threads you want to run concurrently.

### JMX instrumentation

You may wish to monitor the internal operation of your Rundeck server via JMX.

JMX provides introspection on the JVM, the application server,
and classes all through a consistent interface.
These various components are exposed to the management console
via JMX managed beans - MBeans for short.

_Note_: For more background information on JMX, see
"[Java theory and practice: Instrumenting applications with JMX.](https://www.ibm.com/developerworks/library/j-jtp09196/)".

Enable local JMX monitoring by adding the ``com.sun.management.jmxremote``
flag to the startup parameters in the [profile](../configuration/configuration-file-reference.html#profile).

~~~~~ {.bash}
export RDECK_JVM="$RDECK_JVM -Dcom.sun.management.jmxremote"
~~~~~

You use a JMX client to monitor JMX agents.
This can be a desktop GUI like JConsole run locally.

    jconsole <rundeck pid>

For instructions on remote JMX monitoring for Grails, Spring and log4j see:
[Grails in the enterprise](https://public.dhe.ibm.com/software/dw/java/j-grails12168-pdf.pdf).

### Node execution

If you are executing commands across many hundreds or thousands of hosts, the bundled SSH node executor may not meet your performance requirements. Each SSH connection uses multiple threads and the encryption/decryption of messages uses CPU cycles and memory. Depending on your environment, you might choose another Node executor like MCollective, Salt or something similar. This essentially delegates remote execution to another tool designed for asynchronous fan out and thus relieving Rundeck of managing remote task execution.

### Built in SSH plugins

If you are interested in using the built in [SSH plugins](../../manual/node-execution/ssh-node-execution.html), here are some details about how it performs when executing commands across very large numbers of nodes. For these tests, Rundeck was running on an 8 core, 32GB RAM m2.4xlarge AWS EC2 instance.

We chose the `rpm -q` command which checks against the rpm database to see if a particular package was installed.  For 1000 nodes we saw an average execution of 52 seconds.  A 4000 node cluster  took roughly 3.5 minutes, and 8000 node cluster about 7 minutes.

The main limitation appears to be memory of the JVM instance relative to the number of concurrent requests.  We tuned the max memory to be 12GB with a 1000 Concurrent Dispatch Threads to 1GB of Memory.  GC appears to behave well during the runs given the "bursty" nature of them.

### SSL and HTTPS performance

It is possible to offload SSL connection processing by using an SSL termination proxy. This can be accomplished by setting up Apache httpd or [Nginx](https://en.wikipedia.org/wiki/Nginx) as a frontend to your Rundeck instances.

### Resource provider

Rundeck projects obtain information about nodes via a
[resource provider](../configuration/resource-model-sources/index.html). If your resource provider is a long blocking process (due to slow responses from a backend service), it can slow down or even hang up Rundeck. Be sure to make your resource provider work asynchronously.
Also, consider using caching when possible.
