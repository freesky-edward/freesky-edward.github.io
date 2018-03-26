---
layout: post
title:  "How to easily change the version of golang on Ubuntu"
lang: en
date: 2017-11-6
---

### Background Introduction

Currently, Various applications are based on golang in your cloud. when you are trying to debug these applications, you may find that different application requires different version of golang. some requires running on the newest version, some requires fix version, like go 1.6. how to make your environment agile, that is what I am meant to explain here.


### Compile Your Golang From Source

If you want to change the go version flexibly, compile it from source should be one of the optional choice you may think. the official website has given a good start on building go binary from source code. You can directly jump to [Installing Go from source](https://golang.org/doc/install/source) and read it. I am not going to talking much about the basic concept or optional steps on it.

Here we will fix on how to build it on ubuntu and give a hand-by-hand steps. if you want to follow and test it. please prepare the ubuntu server first. my test environment is ubuntu 16.04. 

Now, let's directly move to the steps.    

- update apt-get 

```shell
sudo apt-get update
sudo apt-get upgrade
```

- install git    

check it is installed before installing.

```shell
git version
```

if it returns version number, please jump to step 3 directly. if not, execute the following commands

```shell
sudo apt-get install git
```

- install gcc

```shell
root@SZX1000219582:/home/slob/go_evn# sudo apt-get install gcc   
```

- download go1.4   

```shell
wget https://storage.googleapis.com/golang/go1.4.3.linux-amd64.tar.gz
tar -xvf go1.4.3.linux-amd64.tar.gz -C ./go1.4
```

- configure environments

open the profile of environment

```shell
vim /etc/profile
```
append  the following sentences into the last row

```shell
#replace the <current-path> with the go1.4 source path
export GOROOT_BOOTSTRAP=<current-path>/go1.4   
export CGO_ENABLED=0
```

- check out source

```shell
git clone https://go.googlesource.com/go
```

then checkout the version which you want to install if necessary.

```shell
cd go
checkout go1.9.2
```

- build binary

```shell
cd go/src
./all.bash
```

the **all.bash** will build the source into bin directory and execute the test case. if you don't want to execute test or want to save time, you could run **./make.bash** instead.

- configure the path    

open the profile of environment

```shell
vim /etc/profile
```
append  the following sentences into the last row

```shell
#replace the <current-path> with the go source path
export PATH=<current-path>/go/bin:$PATH
```

and then make it work   

```shell
source /etc/profile  
```

- test if successful    

```shell
go version
```

if the version number was printed, it means all work well.

### Change the version

if you want to change the go version,  only do the step of "check out source" (remember to change the version number as you want ) and step of "build binary". then all that will be ok. if you want to make it more easy. you can write a shell binary to finish that via the given version number as a parameter.

