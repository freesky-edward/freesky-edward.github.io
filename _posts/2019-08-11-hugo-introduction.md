---
layout: post
title: "Hugo Brief Introduction"
date: 2019-08-11
type: tech
---

### Hugo Introduction
Hugo is one of the best website generators and frameworks, with which we can build a website very easy. it is a similar to jekyll while it writes in golang with higher spreed building. [themes markets]([https://themes.gohugo.io/](https://themes.gohugo.io/)) are open to veryone.

![Image title](https://dzone.com/storage/temp/11453293-static-site-generator.jpg)

### Hugo & Jekyll

The following image linked from internet shows us the difference between the two frameworks.

![Static Site Generators](https://dzone.com/storage/temp/11453310-infographic-gatsby-hugo-jekyll.jpg)

### Usage

There are several ways to installation hugo. it would be easy to finish it by following the official [guide](https://gohugo.io/getting-started/installing/). however, as a developer, I would build it from source. 

build from source:
```
go get -v github.com/gohugoio/hugo
echo "PATH=$GOPATH/bin:$PATH" >> /etc/profile
source /etc/profile
```

check:

```
hugo version
Hugo Static Site Generator v0.56.3 linux/amd64 BuildDate: unknown
```

start hugo project:

```
hugo new site test
git submodule add <theme-git-addree> themes/<theme-name>
echo  'theme = "<theme-name>"' >> config.toml
```

Generally, the theme has itself example, the only thing to do is to cope the example folder to theme root folder including config.toml file.

build:
```
hugo
```

the ```public``` folder will be generated when running hugo command. all the website content are located in this folder. it can run by web-server directly.

run by hugo:

```
hugo server --bind 0.0.0.0 -p 1313 --baseURL http://<public-ip>:1313/
```


 
