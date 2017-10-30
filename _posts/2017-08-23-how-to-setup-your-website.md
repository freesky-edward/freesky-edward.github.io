---
layout: post
title: "How To Set Up Your Own Blog Site"
date: 2017-08-23
---

## Backgroud 

I looked for a way to build my own blog site for a while, recently, it took me some time to finish this. here you saw is the achievement. I'd like share all the solution here for who is going to follow this way.

## Contents

There would be many ways you can change, the simplest way is to use github pages. [guides](http://jmcglone.com/guides/github-pages/) might be help you. I chosed a little complex way only because I want to use my own domain name. so, this might not be your best chovice, it's only my interests.

My solution is:

*. Create GCP Instance.
*. Build the web content base on a template.
*. Run web server on GCP cloud.
*. Create a domain name and reference to GCP instance address.

the simplest logic map looks like:
a domain(GoDaddy) --> a web server(Jekyll) --|--> the content(Jekyll theme)
                                             |--> VM instance(GCP)

### Create GCP Instance 

I don't want to talk much about how to get your server instance on GCP, the quick start of GCP give a very detailed view of that, it can be found here: https://cloud.google.com/compute/docs/quickstart-linux. just follow it.

### Build the web content

Before you begin, It's helpful to take a look at how jekyll works. jekyll is text transformation engine that produce the web content via combining the text writing in markup language, etc. markdown, texttile.. and the layout files. it has its own directories definitions. the basic site will looks like:

-
|--_config.yml
|--_data
|  |--XXXX.yml
|--_includes
|  |--head.html
|  |--footer.html
|--_layouts
|  |--default.html
|  |--post.html
|--_posts
|  |--2017-08-31-how-to-setup-your-website.md
|  |--2017-08-31-introduce-myself.md
|--_site
|--index.html

As the above showed,the index.html is the main page of the website. it alway provides the the guide page for the website.
generally it will use _data to config other sub-page, each of sub-page can be a real page under _includes directory or text context under _post directory with layout descriptor under _layouts directory. the actual blog content locates under _posts directory writing by markdown.
for more information [here](https://jekyllrb.com/docs/structure/)

There also have some other folders etc. css, image... you can define whatever as you like. the only thing is to reference them to you web page by html syntax.  

I don't think it's a good idea to setup everything by yourself. otherwise to find a template will save much time. choose one from [here](http://jekyllthemes.org/) and download it.

The left work is to replace the information in each pages according to your favorites. the important one is the _config.yml.

Do not forget to upload your last version into github. usually, create a repository on github first. then clone it on your work space by

```
git clone https://github.com/yourrepository 
```

push your change into github after finish changing.

```
git add .
git commit -m "some comments for this commit"
git push origin/master
```

### Run web server 

I can introduce more detailed on this section... all of the below work is done on GCP instance

1. setup  the jekyll

```
sudo apt-get install ruby-dev
gem install jekyll bundler
```

2. clone your web content

```
git clone https://github.com/yourrepository
```

3. install the gems

```
cd your-repository-folder
bundle install
```

4. run the jekyll under the repository directory

```
bundle exec jekyll serve -H 0.0.0.0 -P 80
```

the main steps is all above, some things you must take. 
1. set the the JEKYLL_ENV=production variable into your bash profile. if not you will find the navigator can not find the css files and image files
2. if you want to run the server in backgroup and forward the output to some file, please use

```
nohup bundle exec jekyll serve -H 0.0.0.0 -P 80 &> /var/log/mywebsit.log &2>1 &
```

anyother error can be fixed by google.

### Create a domain and reference to GCP server

I used [GoDaddy](https://www.godaddy.com/) to regist my domain, it will take you some money. the registion work can be finished easily, the only difficault thing is to find a favourite name without be taken by other.

Change 'points to' the A field into your GCP public ip following [here](https://www.godaddy.com/help/change-my-ip-address-20134).

## Summarize

To build a your own blog site is not difficault as you think. I think the most difficault thing is modifying the web content. rest thing is only labor.
