---
layout: post
title:  "Networking is slow from Windows"
date:   2016-06-01 15:00:00 -0500
categories: Puzzling Observations
---

<style type="text/css">
table{
        border-collapse: collapse;
        border-spacing: 2px; border:2px solid #000000;
        }

th{ border:2px solid #000000; } td{ border:2px solid #000000; padding-right:10px; padding-left:10px} 
</style>

Background
----------
You may have read my previous post on [slow SSH/SCP in windows]({% post_url 2016-05-25-slow-windows-ssh %}). In this post, I will explore this in a bit more detail. As it turns out, the network performance seems to be generally worse in Windows and I am wondering why. In this experiment, I am using Microsoft Azure (where one would think that Windows would be working optimally) to measure network throughput. I have set up a number of clients and servers in both local and remote data centers. I am using [iperf3](https://iperf.fr/) to measure network performance of direct connections and connections established through SSH tunnels. The SSH tunnel scenario is something I am particularly interested in for connecting the clinical MRI systems to the [Gadgetron](https://gadgetron.github.io). 


Experiment
----------
All virtual machines in the following were set up in Microsoft [Azure](https://www.azure.com) and were of type Standard DS3 v2 (4 cores, 14 GB memory). Two servers were established in EASTUS and WESTUS and two clients (one Linux and one Windows) were set up in EASTUS. `iperf3` was used to measure network throughput from both clients to both servers. The throughputs were measured using direct connections and connecting through an SSH tunnel established from the client to the server. On Windows, OpenSSH in Cygwin was used to establish the tunnels. The network topology is outlined in the diagram below.

{% highlight text  %}

                      EASTUS

+---------------------------------------------------------------+
|                                                               |
|                                        Linux Client           |                       WESTUS
|                                     +-----------------+       |
|        Linux Server                 |                 |       |        +-----------------------------+
|     +----------------+     +--------+  Ubuntu 14.04   +-------------+  |                             |
|     |                |     |        | 40.117.152.122  |       |     |  |        Linux Server         |
|     |  Ubuntu 14.04  |     |        |                 |       |     |  |     +----------------+      |
|     |                <-----+        +-----------------+       |     +-------->                |      |
|     |  13.82.49.53   |                                        |        |     |  Ubuntu 14.04  |      |
|     |                |                 Windows Client         |        |     |  13.88.182.107 |      |
|     |                <-----+        +-----------------+       |     +-------->                |      |
|     |                |     |        |                 |       |     |  |     +----------------+      |
|     +----------------+     |        | Win Serv 2012R2 |       |     |  |                             |
|                            +--------+  13.90.249.191  +-------------+  |                             |
|                                     |                 |       |        +-----------------------------+
|                                     +-----------------+       |
|                                                               |
+---------------------------------------------------------------+
{% endhighlight %}

Results and Dicussion
---------------------

The full results of the of the `iperf3` runs are listed at the end of this post. A summary of the results can be seen in the table below. 


|       | EASTUS (Direct) | EASTUS (SSH)    | WESTUS (DIRECT)  | WESTUS (SSH)     |
|-------|-----------------|-----------------|------------------|------------------|
| LINUX | 236 MB/s (100%) | 205 MB/s (100%) | 44.9 MB/s (100%) | 15.6 MB/s (100%) |
| WIN   | 209 MB/s (89%)  | 115 MB/s (56%)  | 3.65 MB/s (8%)   | 3.8 MB/s (24%)   |
|-------|-----------------|-----------------|------------------|------------------|


It can be seen that in all instances network performance is poorer when using the Windows client. Even for a local (within EASTUS) connection, there is a drop in performance. In case you are thinking that the Linux machine could be located more favorably compared to the server, that is true, but I have tried this experiment a number of times (with different instances both in Azure and on a local network) and the result seems to always be the same. Windows just performs worse. When going through a tunnel, the problems get worse. There is almost a 50% drop in performance locally and much more when going long distance.

I struggle to explain these results and would appreciate any hints/comments that anybody might have. Especially good suggestions for fixing it. This must be a serious concern for anybody working with large datasets on Windows?

Full Data
---------

Connecting directly Windows Client (EASTUS) to Linux Server (EASTUS):

{% highlight shell  %}
C:\Users\gadgetron\Downloads\iperf-3.1.2-win64>iperf3 -c 13.82.49.53 -p 5207 -f MG
Connecting to host 13.82.49.53, port 5207
[  4] local 10.0.0.5 port 49350 connected to 13.82.49.53 port 5207
[ ID] Interval           Transfer     Bandwidth
[  4]   0.00-1.00   sec   194 MBytes   194 MBytes/sec
[  4]   1.00-2.00   sec   214 MBytes   213 MBytes/sec
[  4]   2.00-3.00   sec   220 MBytes   220 MBytes/sec
[  4]   3.00-4.00   sec   207 MBytes   207 MBytes/sec
[  4]   4.00-5.00   sec   206 MBytes   206 MBytes/sec
[  4]   5.00-6.00   sec   211 MBytes   211 MBytes/sec
[  4]   6.00-7.00   sec   207 MBytes   207 MBytes/sec
[  4]   7.00-8.00   sec   215 MBytes   215 MBytes/sec
[  4]   8.00-9.00   sec   210 MBytes   210 MBytes/sec
[  4]   9.00-10.00  sec   210 MBytes   210 MBytes/sec
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bandwidth
[  4]   0.00-10.00  sec  2.04 GBytes   209 MBytes/sec                  sender
[  4]   0.00-10.00  sec  2.04 GBytes   209 MBytes/sec                  receiver

iperf Done.
{% endhighlight %}

Connecting directly Linux Client (EASTUS) to Linux Server (EASTUS):

{% highlight bash %}
gadgetron@linuxClient2:~$ iperf3 -c 13.82.49.53 -p 5207 -f MG
Connecting to host 13.82.49.53, port 5207
[  4] local 10.0.0.7 port 47544 connected to 13.82.49.53 port 5207
[ ID] Interval           Transfer     Bandwidth       Retr  Cwnd
[  4]   0.00-1.00   sec   217 MBytes   217 MBytes/sec    0   2.09 MBytes
[  4]   1.00-2.00   sec   239 MBytes   239 MBytes/sec    0   2.28 MBytes
[  4]   2.00-3.00   sec   238 MBytes   237 MBytes/sec    0   2.28 MBytes
[  4]   3.00-4.00   sec   238 MBytes   237 MBytes/sec    0   2.28 MBytes
[  4]   4.00-5.00   sec   238 MBytes   238 MBytes/sec    0   2.28 MBytes
[  4]   5.00-6.00   sec   239 MBytes   239 MBytes/sec    0   2.28 MBytes
[  4]   6.00-7.00   sec   238 MBytes   237 MBytes/sec    0   2.28 MBytes
[  4]   7.00-8.00   sec   240 MBytes   240 MBytes/sec  219   1.74 MBytes
[  4]   8.00-9.00   sec   236 MBytes   236 MBytes/sec    0   1.74 MBytes
[  4]   9.00-10.00  sec   240 MBytes   240 MBytes/sec    0   1.74 MBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bandwidth       Retr
[  4]   0.00-10.00  sec  2.31 GBytes   236 MBytes/sec  219             sender
[  4]   0.00-10.00  sec  2.30 GBytes   236 MBytes/sec                  receiver

iperf Done.
{% endhighlight %}

Connecting directly Windows Client (EASTUS) to Linux Server (WESTUS):

{% highlight shell %}
C:\Users\gadgetron\Downloads\iperf-3.1.2-win64>iperf3 -c 13.88.182.107 -p 5207 -f MG
Connecting to host 13.88.182.107, port 5207
[  4] local 10.0.0.5 port 49354 connected to 13.88.182.107 port 5207
[ ID] Interval           Transfer     Bandwidth
[  4]   0.00-1.00   sec  2.25 MBytes  2.25 MBytes/sec
[  4]   1.00-2.01   sec  3.88 MBytes  3.85 MBytes/sec
[  4]   2.01-3.01   sec  3.88 MBytes  3.85 MBytes/sec
[  4]   3.01-4.01   sec  3.75 MBytes  3.75 MBytes/sec
[  4]   4.01-5.00   sec  3.75 MBytes  3.79 MBytes/sec
[  4]   5.00-6.00   sec  3.75 MBytes  3.74 MBytes/sec
[  4]   6.00-7.00   sec  3.75 MBytes  3.75 MBytes/sec
[  4]   7.00-8.00   sec  4.00 MBytes  4.00 MBytes/sec
[  4]   8.00-9.01   sec  3.75 MBytes  3.72 MBytes/sec
[  4]   9.01-10.00  sec  3.75 MBytes  3.80 MBytes/sec
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bandwidth
[  4]   0.00-10.00  sec  36.5 MBytes  3.65 MBytes/sec                  sender
[  4]   0.00-10.00  sec  36.5 MBytes  3.65 MBytes/sec                  receiver

iperf Done.
{% endhighlight %}

Connecting directly Linux Client (EASTUS) to Linux Server (WESTUS):

{% highlight bash %}
gadgetron@linuxClient2:~$ iperf3 -c 13.88.182.107 -p 5207 -f MG
Connecting to host 13.88.182.107, port 5207
[  4] local 10.0.0.7 port 53653 connected to 13.88.182.107 port 5207
[ ID] Interval           Transfer     Bandwidth       Retr  Cwnd
[  4]   0.00-1.00   sec  25.2 MBytes  25.2 MBytes/sec    0   6.00 MBytes
[  4]   1.00-2.00   sec  47.5 MBytes  47.5 MBytes/sec    0   6.00 MBytes
[  4]   2.00-3.00   sec  47.5 MBytes  47.5 MBytes/sec    0   6.00 MBytes
[  4]   3.00-4.00   sec  46.2 MBytes  46.2 MBytes/sec    0   6.00 MBytes
[  4]   4.00-5.00   sec  46.2 MBytes  46.2 MBytes/sec    0   6.00 MBytes
[  4]   5.00-6.00   sec  47.5 MBytes  47.5 MBytes/sec    0   6.00 MBytes
[  4]   6.00-7.00   sec  47.5 MBytes  47.5 MBytes/sec    0   6.00 MBytes
[  4]   7.00-8.00   sec  46.2 MBytes  46.2 MBytes/sec    0   6.00 MBytes
[  4]   8.00-9.00   sec  47.5 MBytes  47.5 MBytes/sec    0   6.00 MBytes
[  4]   9.00-10.00  sec  47.5 MBytes  47.5 MBytes/sec    0   6.00 MBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bandwidth       Retr
[  4]   0.00-10.00  sec   449 MBytes  44.9 MBytes/sec    0             sender
[  4]   0.00-10.00  sec   449 MBytes  44.9 MBytes/sec                  receiver

iperf Done.
{% endhighlight %}

SSH tunnel connection Linux Client (EASTUS) to Linux Server (EASTUS):

{% highlight bash %}
gadgetron@linuxClient2:~$ iperf3 -c localhost -p 5207 -f MG
Connecting to host localhost, port 5207
[  4] local 127.0.0.1 port 50700 connected to 127.0.0.1 port 5207
[ ID] Interval           Transfer     Bandwidth       Retr  Cwnd
[  4]   0.00-1.00   sec   186 MBytes   186 MBytes/sec    0    639 KBytes
[  4]   1.00-2.00   sec   201 MBytes   201 MBytes/sec    0    639 KBytes
[  4]   2.00-3.00   sec   205 MBytes   205 MBytes/sec    0    639 KBytes
[  4]   3.00-4.00   sec   204 MBytes   204 MBytes/sec    0    639 KBytes
[  4]   4.00-5.00   sec   209 MBytes   209 MBytes/sec    0    639 KBytes
[  4]   5.00-6.00   sec   207 MBytes   207 MBytes/sec    0    639 KBytes
[  4]   6.00-7.00   sec   208 MBytes   208 MBytes/sec    0    639 KBytes
[  4]   7.00-8.00   sec   211 MBytes   211 MBytes/sec    0    639 KBytes
[  4]   8.00-9.00   sec   210 MBytes   210 MBytes/sec    0    639 KBytes
[  4]   9.00-10.00  sec   208 MBytes   208 MBytes/sec    0    639 KBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bandwidth       Retr
[  4]   0.00-10.00  sec  2.00 GBytes   205 MBytes/sec    0             sender
[  4]   0.00-10.00  sec  1.99 GBytes   204 MBytes/sec                  receiver

iperf Done.
{% endhighlight %}

SSH tunnel Windows Client (EASTUS) to Linux Server (EASTUS):

{% highlight shell %}
C:\Users\gadgetron\Downloads\iperf-3.1.2-win64>iperf3 -c localhost -p 5207 -f MG

Connecting to host localhost, port 5207
[  4] local ::1 port 49361 connected to ::1 port 5207
[ ID] Interval           Transfer     Bandwidth
[  4]   0.00-1.00   sec   117 MBytes   117 MBytes/sec
[  4]   1.00-2.00   sec   113 MBytes   113 MBytes/sec
[  4]   2.00-3.00   sec   116 MBytes   116 MBytes/sec
[  4]   3.00-4.00   sec   117 MBytes   117 MBytes/sec
[  4]   4.00-5.00   sec   115 MBytes   115 MBytes/sec
[  4]   5.00-6.00   sec   100 MBytes   100 MBytes/sec
[  4]   6.00-7.00   sec  99.9 MBytes  99.9 MBytes/sec
[  4]   7.00-8.00   sec   116 MBytes   116 MBytes/sec
[  4]   8.00-9.00   sec   115 MBytes   115 MBytes/sec
[  4]   9.00-10.00  sec   118 MBytes   117 MBytes/sec
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bandwidth
[  4]   0.00-10.00  sec  1.10 GBytes   113 MBytes/sec                  sender
[  4]   0.00-10.00  sec  1.10 GBytes   113 MBytes/sec                  receiver

iperf Done.
{% endhighlight %}

SSH tunnel Linux Client (EASTUS) to Linux Server (WESTUS):

{% highlight bash %}
gadgetron@linuxClient2:~$ iperf3 -c localhost -p 5207 -f MG
Connecting to host localhost, port 5207
[  4] local 127.0.0.1 port 50745 connected to 127.0.0.1 port 5207
[ ID] Interval           Transfer     Bandwidth       Retr  Cwnd
[  4]   0.00-1.00   sec  20.0 MBytes  20.0 MBytes/sec    0   2.62 MBytes
[  4]   1.00-2.00   sec  15.0 MBytes  15.0 MBytes/sec    0   2.62 MBytes
[  4]   2.00-3.00   sec  15.0 MBytes  15.0 MBytes/sec    0   2.62 MBytes
[  4]   3.00-4.00   sec  15.0 MBytes  15.0 MBytes/sec    0   2.62 MBytes
[  4]   4.00-5.00   sec  15.0 MBytes  15.0 MBytes/sec    0   2.62 MBytes
[  4]   5.00-6.00   sec  15.0 MBytes  15.0 MBytes/sec    0   2.62 MBytes
[  4]   6.00-7.00   sec  16.2 MBytes  16.2 MBytes/sec    0   2.62 MBytes
[  4]   7.00-8.00   sec  15.0 MBytes  15.0 MBytes/sec    0   2.62 MBytes
[  4]   8.00-9.00   sec  13.8 MBytes  13.7 MBytes/sec    0   2.62 MBytes
[  4]   9.00-10.00  sec  16.2 MBytes  16.3 MBytes/sec    0   2.62 MBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bandwidth       Retr
[  4]   0.00-10.00  sec   156 MBytes  15.6 MBytes/sec    0             sender
[  4]   0.00-10.00  sec   147 MBytes  14.7 MBytes/sec                  receiver

iperf Done.
{% endhighlight %}

SSH tunnel Windows Client (EASTUS) to Linux Server (WESTUS):

{% highlight shell %}
C:\Users\gadgetron\Downloads\iperf-3.1.2-win64>iperf3 -c localhost -p 5207 -f MG

Connecting to host localhost, port 5207
[  4] local ::1 port 49366 connected to ::1 port 5207
[ ID] Interval           Transfer     Bandwidth
[  4]   0.00-1.00   sec  6.12 MBytes  6.10 MBytes/sec
[  4]   1.00-2.01   sec  3.50 MBytes  3.49 MBytes/sec
[  4]   2.01-3.01   sec  3.50 MBytes  3.47 MBytes/sec
[  4]   3.01-4.00   sec  3.50 MBytes  3.53 MBytes/sec
[  4]   4.00-5.00   sec  3.50 MBytes  3.50 MBytes/sec
[  4]   5.00-6.01   sec  3.75 MBytes  3.74 MBytes/sec
[  4]   6.01-7.01   sec  3.50 MBytes  3.50 MBytes/sec
[  4]   7.01-8.00   sec  3.38 MBytes  3.39 MBytes/sec
[  4]   8.00-9.01   sec  3.75 MBytes  3.73 MBytes/sec
[  4]   9.01-10.01  sec  3.50 MBytes  3.49 MBytes/sec
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bandwidth
[  4]   0.00-10.01  sec  38.0 MBytes  3.80 MBytes/sec                  sender
[  4]   0.00-10.01  sec  35.6 MBytes  3.55 MBytes/sec                  receiver

iperf Done.
{% endhighlight %}