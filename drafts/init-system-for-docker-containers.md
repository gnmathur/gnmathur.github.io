+++
authors = ["Gaurav Mathur"]
title = "Improved init for Docker containers"
date = "2023-11-11"
description = "Using a better init to allow better handling for zombie processes"
tags = [
    "docker",
    "unix",
    "pid 1",
    "patterns",
]
categories = [
    "docker"
]
series = ["Docker"]
+++

# Introduction

This article describes a way to reason about zombie processes in a Docker container environment, and a suggested approach
to solve the problem.

# Problem

Let's start with an application that has to run in a Docker container. One of the primary requirements for this
application would be handle graceful exits. So the application inside the container has to handle signals properly.
That means that the application has to be PID 1. PID 1 has a special significance inside containers. Among other
things, PID 1 is responsible for handling system signals sent to the container. For example, when you stop a container,
Docker sends a SIGTERM signal to PID 1. The application would typically handle this signal to shut down gracefully.

So, we start with a natural or recommended way of creating a container where we point the Docker entrypoint to the
application, and if the application has some preable or startup script associated with it, making sure that we `exec`
the application at the end of that script.

Now assume that we have an additional requirement to automatically provision something in this application after it starts
up. For that we need another script or program to start, alongside this application, which runs the provisioning steps after
the application starts up. Docker containers typically have a single entry point, and arguably that's the most common design
pattern. However, in this case we have a need to have _two_ entrypoints - one the applicaiton, and two the script that
provisions the application post startup. The important thing is that the script has a natural completion point - it completes
and natually exists after doing its job.

Let's write this application first in Python.

```python
import time
import errno                   
import signal                  
import sys

cmd_pipe = '/tmp/cmd_pipe'
ack_pipe = '/tmp/ack_pipe'     
    
def try_provisioning():        
    try:
        message = reader.readline().strip()
        if message:
            print(f"[App] Received message: {message}")
            with open(ack_pipe, 'w') as writer:
                writer.write('ack\n')           
            return True
    except OSError as e:
        if e.errno != errno.EAGAIN and e.errno != errno.EWOULDBLOCK:
            pass # Ignore errors for our purposes
    return False

# Simulate graceful exit
def sigterm_handler(_signo, _stack_frame):
    print("[App] SIGTERM received, shutting down...")
    sys.exit(0)

# Handle SIGTERM
signal.signal(signal.SIGTERM, sigterm_handler)
signal.signal(signal.SIGINT, sigterm_handler)

print("[App] Starting the application...")

with open(cmd_pipe, 'r') as reader:
    fd = reader.fileno()
    os.set_blocking(fd, False)  # Set the pipe to non-blocking  
    provisioned = False

    while True:
        # Provision once
        if not provisioned:
            provisioned = try_provisioning()

        # Simulate work
        time.sleep(2)
```

This application waits for a "provisioning" action on a pipe `/tmp/cmd_pipe`. Once it receives that, it does some processing, and sends an acknowledgement to the provisioner that its done.

Let's try writing this provisioning application/script next.

```python
import os
import sys                     

cmd_pipe = '/tmp/cmd_pipe'
ack_pipe = '/tmp/ack_pipe'     
  
message = "Provision this"

with open(cmd_pipe, 'w') as pipe:
    pipe.write(message + '\n') 
    print("Message sent. Waiting for acknowledgment...")
  
with open(ack_pipe, 'r') as ack_pipe:
    ack = ack_pipe.readline().strip()
    if ack:
        print("Acknowledgment received. Exiting...")
```

Finally, let's wrap it all together in a Docker container. First, let's define the Docker entrypoint script. 

```bash
#!/bin/bash
                               
# Start the provisioner in the background
python3 /app/provision.py &    
   
# Start the application
exec python3 /app/app.py   
```

Notice, that we `exec` the application. That's because we want it to run as PID 1 in the container.

```Dockerfile
FROM python:3.9-slim           

RUN apt-get update && apt-get install -y procps && rm -rf /var/lib/apt/lists/*
  
WORKDIR /app
  
RUN mkfifo /tmp/cmd_pipe       
RUN mkfifo /tmp/ack_pipe
  
COPY app.py /app/app.py
COPY provision.py /app/provision.py
COPY docker-entrypoint.sh /app/docker-entrypoint.sh

RUN chmod +x /app/docker-entrypoint.sh

# Start both the application and the provisioner
ENTRYPOINT ["/app/docker-entrypoint.sh"]
```

Now that we have put this all together, let's build and run the container.

```bash
>> docker build . -t app:0.1
```

```bash
>> docker run --name myapp -it app:0.1
[App] Starting the application...
[Provisioner] Provisioning message sent. Waiting for acknowledgment...
[App] Received message: provision
[Provisioner] Acknowledgment received. Exiting...
```

At this point the application stays in the busy loop forever, simulating work after provisioning.

Now for the interesting bit and to arrive at the main point of this article. If we peek inside the container and use the `ps` command
to example the process table, we'll see the state like this -

```bash
> docker exec -it myapp /bin/bash
root@3f7598bb8611:/app# ps -eafww
UID          PID    PPID  C STIME TTY          TIME CMD
root           1       0  1 05:14 pts/0    00:00:00 python3 /app/app.py
root           7       1  0 05:14 pts/0    00:00:00 [python3] <defunct>
root           8       0  2 05:14 pts/1    00:00:00 /bin/bash
root          14       8 66 05:14 pts/1    00:00:00 ps -eafww
```

The _defunct_ process if of interest to us. This is a process that was the provisioner Python script, that did its job and
exited, but it ***got left behind a zombie process***.

So why is there a zombie process at all? In this use case the container exec'd into the application and it become PID 1. As we
noted before, PID 1 has a special place in a Docker container (and really in Unix system in general). PID 1 has the special
responsibility of "reaping" zombie processes. These are child processes that have completed execution but still remain in the
process table, usually because their parent hasn't read their exit status. However, in our use case here, the application did
not start the provisioning script as a child process. That meant that it had no ability to handle SIGCHLD, or firsly even
recognizing the provisioner application at all. However, since the appication was exec'd into by the `docker-entrypoin.sh`, it
became the process with PID 1 but had not ability to handle the exit of the provisioner. That meant that the provisioner was
left as a zombie. This is a problem. However, for this use case its not such a huge problem. Its not such a huge deal that we
have one zombie process lying around for the duration of the containers uptime, though it does look ugly, and more importantly, it might be a hint to use that _we might not be designing the container in the best possible way_.

Consider another example of an application that spawns multiple child processes but does not handle their exit properly, and
create a Dockerfile for it.

```python
import os, signal, time

# Simulate graceful exit
def sigterm_handler(_signo, _stack_frame):
    print("[App] SIGTERM received, shutting down...")
    sys.exit(0)
    
# Handle SIGTERM
signal.signal(signal.SIGTERM, sigterm_handler)
signal.signal(signal.SIGINT, sigterm_handler)
    
def main():
    for _ in range(10):        
        pid = os.fork()
        if pid == 0:
            # Child process
            print(f"Child process {os.getpid()} doing some work.")
            time.sleep(1)  # Simulate some work
            print(f"Child process {os.getpid()} exiting.")
            os._exit(0)
        else:
            # Parent process continues without waiting
            continue

    print("Parent process finished creating children, now doing its own work.")

    while True:
        time.sleep(30)  # Simulate parent's own work

    print("Parent process exiting.")

if __name__ == "__main__":
    main()
```

Let's create a container and run the application.

```bash
>> docker run --name myapp -it app:0.1
Child process 7 doing some work.
Child process 8 doing some work.
Child process 9 doing some work.
Child process 10 doing some work.
Child process 11 doing some work.
Child process 12 doing some work.
Child process 13 doing some work.
Child process 14 doing some work.
Child process 15 doing some work.
Parent process finished creating children, now doing its own work.
Child process 16 doing some work.
Child process 9 exiting.
Child process 7 exiting.
Child process 8 exiting.
Child process 10 exiting.
Child process 13 exiting.
Child process 11 exiting.
Child process 12 exiting.
Child process 15 exiting.
Child process 14 exiting.
Child process 16 exiting.
```

The application started 10 child process, which started and did their job an exited. Now let's take a peek at the container
process table and see what that looks like. 

```bash
>> docker exec -it myapp /bin/bash
root@dd1c26a7a9cb:/app# ps -eaf
UID          PID    PPID  C STIME TTY          TIME CMD
root           1       0  0 07:36 pts/0    00:00:00 python3 /app/child-spawner.py
root           7       1  0 07:36 pts/0    00:00:00 [python3] <defunct>
root           8       1  0 07:36 pts/0    00:00:00 [python3] <defunct>
root           9       1  0 07:36 pts/0    00:00:00 [python3] <defunct>
root          10       1  0 07:36 pts/0    00:00:00 [python3] <defunct>
root          11       1  0 07:36 pts/0    00:00:00 [python3] <defunct>
root          12       1  0 07:36 pts/0    00:00:00 [python3] <defunct>
root          13       1  0 07:36 pts/0    00:00:00 [python3] <defunct>
root          14       1  0 07:36 pts/0    00:00:00 [python3] <defunct>
root          15       1  0 07:36 pts/0    00:00:00 [python3] <defunct>
root          16       1  0 07:36 pts/0    00:00:00 [python3] <defunct>
root          27       0  2 07:38 pts/1    00:00:00 /bin/bash
root          33      27  0 07:38 pts/1    00:00:00 ps -eaf
root@dd1c26a7a9cb:/app# 
```

Here again, we notice the same problem - because we had an application that does not properly handle exit of a child
processes, we end up with a lot of zombie processes. If the application is creating these processes regularly, ***there is a
real risk of resource exhaustion***.

> there are multiple approaches to write applications the _correct_ way, to handle child exits properly and not
be in a situation where a child is not reaped properly, but that's not the focus of the article. For reference, explore the
__double-fork technique__ as one way of solving this issue in application that create _daemon_ processes.

# Solution



When we want to shutdown the container, we can do so by running `docker stop <container_id>. For example,

```bash
>> docker stop myapp
```

You will see the following message on the console

```bash

[App] SIGTERM received, shutting down...
```
