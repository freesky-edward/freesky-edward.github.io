---
layout: post
date: 2017-08-28
title: Setup Golang From Source
url: /build-golang-env-from-master
---

You may guess the main popurse of what I plan to say from the title,I think the main case is that if 
you are asked to test your system can run well on different version of golang, it will be the matter
you have to deal with.

If you expect all your apps can change the golang verison in an easy way.or you want to know how to 
build golang from source,here it is, my example is running on unbutu.

### preparing work 

* install git and gcc

```
sudo apt-get install git
sudo apt-get isntall gcc
```

* download the go 1.4 as the bootstrap

```
wget https://storage.googleapis.com/golang/go1.4.3.linux-amd64.tar.gz
tar -xvf go1.4.3.linux-amd64.tar.gz -C /opt/go1.4
```

* set the environment variable in /etc/profile

```
echo "export GOROOT_BOOTSTRAP=/home/go1.4" >> /etc/profile
```

### checkout the golang source 

There are two repositories can be chosen, one is https://go.googlesource.com/go which is the recommended repository on official website. another is github https://github.com/golang/go. anyone is ok.

```
git clone https://github.com/golang/go /opt
```

### compile the source 

```
cd /opt/go/src
./all.bash
```

There are some things we'd to know, serveral scripts you can see under the source directory(/opt/go/src) if you want to build the source code and run all the test cases, all.bash is the right one as the example above. however,sometimes you might not need to run the test cases, then run the 'make.bash' may save your time.

### get it worked

After compile the source, the binary file of golang has been produced under the bin sub directory of go, however, it doesn't work well when you running the 'go' command. that because the system cann't find the binary. so set the PATH envroment is neccessary.

```
echo "export PATH=/opt/go/bin:$PATH" >> /etc/profile
```

Util now, all that work is finished, swith the version is very simple now. what you should do is just to switch the git branch as you expected and build it again.
