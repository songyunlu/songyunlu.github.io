---
layout: post
title:  Using pssh to Contorl Multiple Machines at the Smae Time
date:   2017-02-21T12:40:00+8:00
---

Never heard of `pssh` before and recently found it's quite useful and convenient for configuring a small group of machines.

### Installation

```bash
# ubuntu as the os
$ sudo apt-get update
$ sudo apt-get install python-pip
$ pip install pssh
```

### Usage

To use the keyfile with pssh, you need to add the file `~/.ssh/config` with host information.

```bash
# for the machines on aws
Host *.compute.amazonaws.com
    IdentityFile ~/.ssh/$KEY_FILE
```

The wildcard `*` here matches to all subdomain of AWS machines. You can specify more host/keyfile pair as you want.

Then you'll speicfy the hosts you want to access to.

```bash
# host.txt
ubuntu@ec2-54-249-57-184.ap-northeast-1.compute.amazonaws.com
ubuntu@ec2-54-238-227-77.ap-northeast-1.compute.amazonaws.com
```

OK, let's give it a try.

```bash

$ pssh --help

# Usage: pssh [OPTIONS] command [...]
# 
# Options:
#   --version             show program's version number and exit
#   --help                show this help message and exit
#   -h HOST_FILE, --hosts=HOST_FILE
#                         hosts file (each line "[user@]host[:port]")
#   -H HOST_STRING, --host=HOST_STRING
#                         additional host entries ("[user@]host[:port]")
#   -l USER, --user=USER  username (OPTIONAL)
#   -p PAR, --par=PAR     max number of parallel threads (OPTIONAL)
#   -o OUTDIR, --outdir=OUTDIR
#                         output directory for stdout files (OPTIONAL)
#   -e ERRDIR, --errdir=ERRDIR
#                         output directory for stderr files (OPTIONAL)
#   -t TIMEOUT, --timeout=TIMEOUT
#                         timeout (secs) (0 = no timeout) per host (OPTIONAL)
#   -O OPTION, --option=OPTION
#                         SSH option (OPTIONAL)
#   -v, --verbose         turn on warning and diagnostic messages (OPTIONAL)
#   -A, --askpass         Ask for a password (OPTIONAL)
#   -x ARGS, --extra-args=ARGS
#                         Extra command-line arguments, with processing for
#                         spaces, quotes, and backslashes
#   -X ARG, --extra-arg=ARG
#                         Extra command-line argument
#   -i, --inline          inline aggregated output and error for each server
#   --inline-stdout       inline standard output for each server
#   -I, --send-input      read from standard input and send as input to ssh
#   -P, --print           print output as we get it
# 
# Example: pssh -h hosts.txt -l irb2 -o /tmp/foo uptime

$ pssh -i -h host.text date

# [1] 16:42:00 [SUCCESS] ubuntu@ec2-54-249-57-184.ap-northeast-1.compute.amazonaws.com
# Tue Feb 21 08:42:00 UTC 2017
# [2] 16:42:00 [SUCCESS] ubuntu@ec2-54-238-227-77.ap-northeast-1.compute.amazonaws.com
# Tue Feb 21 08:42:00 UTC 2017

$ pssh -i -h host.txt 'echo "Hello World!" > hello.txt'

# [1] 16:52:40 [SUCCESS] ubuntu@ec2-54-238-227-77.ap-northeast-1.compute.amazonaws.com
# [2] 16:52:40 [SUCCESS] ubuntu@ec2-54-249-57-184.ap-northeast-1.compute.amazonaws.com
```

Parallel **scp** and **rsync** are also supported.

```bash
# transfer the file to all remote machine.
$ pscp -h host.txt -l ubuntu sample.txt /home/ubuntu/sample.txt

# sync the file between local and remote machines.
$ prsync -h host.txt -l ubuntu sample.txt /home/ubuntu/sample.txt
```

pssh is easy to use for a small number of machines. If are going to control a large fleet of machines, configuration management tools like Chef, Puppet, or Ansilbe are the right tools you shold look at.
