---
layout: post
title:  "SSH/SCP is slow from Windows"
date:   2016-03-12 15:00:00 -0500
categories: Puzzling Observations
---

As you may know I spend a lot of time on the [Gadgetron](https://gadgetron.github.io). One common way of connecting to the Gadgetron from a clinical MRI system is through an SSH tunnel. This has many advantages:

1. Often the MRI scanner component producing the data for the Gadgetron does not have a proper (external) network connection and it is necessary to relay through some other system component (such as the host computer) to get to a network.
2. It ensures that the traffic is encrypted. Normally data is never sent with any patient identifiers (PII/PHI) but as an extra secondary precaution, it is nice to know that the transmission is encrypted.
3. The remote server running the Gadgetron really only needs to have port 22 open in order for the client to connect, so it is easier to manage from a security point of view.

Clearly these features are valuable when running clinical scans using the Gadgetron or when using cloud computing with the Gadgetron.

In practice tunnels are often opened from the MRI system host. One problem that we have encountered is that SSH tunnels from Windows hosts (such as on the Siemens MRI systems) are slow compared to tunnels from a Linux machine. So much so that we often use a Linux machine next to the scanner to do the tunneling to get proper performance. I initially thought this was related to specific configurations on the MRI systems but I decided to do some more investigating.

To explore this a bit further, I set up an SSH server on Azure in their EASTUS data center. I then set up two clients in Azure's EASTUS2 data center (to get some meaningful distance between the client and the server. The two clients were a Linux (Ubuntu 14.04) machine and a Windows (Server 2012 R2) machine. All of the VMs (server and clients) were of type Standard DS3 v2 (4 cores, 14 GB memory). The topology looked something like this:

{% highlight text  %}
     Azure EASTUS2                                Azure EASTUS

+--------------------------------+           +------------------------------+
|       40.79.37.166             |           |                              |
|   +------------------------+   |           |        23.96.60.136          |
|   |                        |   |           |    +--------------------+    |
|   |  Ubuntu 14.04          +----------+    |    |                    |    |
|   |  OpenSSH_6.6.1p1       |   |      |    |    |  Ubuntu 14.04      |    |
|   |                        |   |      |    |    |                    |    |
|   +------------------------+   |      |    |    |  OpenSSH_6.6.1p1   |    |
|                                |      +--------->                    |    |
|       40.79.80.136             |           |    |                    |    |
|   +------------------------+   |           |    |                    |    |
|   |                        |   |      +--------->                    |    |
|   |  Win Server 2012R2     |   |      |    |    +--------------------+    |
|   |                        +----------+    |                              |
|   |                        |   |           |                              |
|   +------------------------+   |           +------------------------------+
|                                |
+--------------------------------+
{% endhighlight %}

I then copied a large file (1432MB) to both clients and attempted to copy it to the server. On the Linux client I just used the standard SCP client installed and on the Windows client I tried:

1. OpenSSH in Cygwin
2. Putty SCP
3. Bitwise sftp client

Let's first look at the transfer from the Linux client to the Linux server:

{% highlight shell  %}
gadgetron@SSHclient:~$ scp 20150923_11h51m34s_62440.dat 23.96.60.136:linux.dat
gadgetron@23.96.60.136's password:
20150923_11h51m34s_62440.dat                     100% 1432MB 204.5MB/s   00:07
{% endhighlight %}

Clearly there is a very good connection (as expected) from EASTUS2 to EASTUS and we are getting over 200MB/s transfer speed. Now let's try the Windows client. We will start with  OpenSSH in Cygwin:

{% highlight shell  %}
gadgetron@WindowsClient ~
$ scp 20150923_11h51m34s_62440.dat 23.96.60.136:win.dat
gadgetron@23.96.60.136's password:
20150923_11h51m34s_62440.dat                     100% 1432MB   7.6MB/s   03:09
{% endhighlight %}

It is basically pathetic performance. Far beyond any fluctuations due to network performance. Let's try with PuTTY instead:

{% highlight bash  %}
C:\Users\gadgetron\Downloads>pscp 20150923_11h51m34s_62440.dat 23.96.60.136:putty.dat
gadgetron@23.96.60.136's password:
20150923_11h51m34s_62440. | 1465908 kB | 31189.5 kB/s | ETA: 00:00:00 | 100%
{% endhighlight %}

Better, but still almost an order of magnitude off the mark. This seems very strange indeed. Now let's try the Bitwise client, which is supposedly the fastest SCP client on Windows:

![Bitwise Download from Windows]({{ site.url }}/assets/bitwise_download_windows.png)

Marginally better than PuTTY, but still way, way off from the Linux performance.

If anybody can tell me what is going on here, I would greatly appreciate it. 




