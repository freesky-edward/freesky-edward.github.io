---
layout: post
lang: en
title: "How To Set Up Your Own Blog Site"
date: 2017-09-06
---

## Brief

As you can see, here is my blog site built by myself, I'd like introduce the general solution of mine,so that you can try out, but I'm not going to try to give you the detailed step operations. you can follow some referenced address to get that.

## Agenda

There would be many ways you can take, the simplest way is to use github pages. [guides](http://jmcglone.com/guides/github-pages/) might to help you. I chose a little complex way only because I want to use my own domain name. so, this might not be your best choice, it's only my interests.

My solution is:

*. Create GCP Instance.
*. Build the web content base on a template.
*. Run the web server on GCP cloud.
*. Create a domain name and link to GCP instance address.

the simplest logic map looks like:
a domain(GoDaddy) --> a web server(Jekyll) --|--> the content(Jekyll theme)     
                                             |--> VM instance(GCP)       

### Create GCP Instance 

I don't want to talk much about how to get your server instance on GCP, the document on google site will give a very clear view of that, click [her](https://cloud.google.com/compute/docs/quickstart-linux). 

### Build the web content

It's helpful to take a look at how jekyll works before your start. jekyll is text transformation engine that produce the web content via combining the text writing in markup language e.g. markdown, texttile.. with the layout files. which has its own directories definitions. the basic structure will looks like:

--
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

As the above showed,the index.html is the main page of the website. it alway provides the the guide page for your website.
generally  _data directory will be used to config your other sub-page, each of sub-page can be a real page under _includes directory or text context under _post directory with layout descriptor under _layouts directory. 
generally, the blogs are under _posts directory which writing by markdown.
for more information [here](https://jekyllrb.com/docs/structure/)

There also have some other folders e.g. css, image... you can define whatever you like. the only concerned thing is to and the link element in your web page by html syntax.  

I don't think it's a good idea to build everything by yourself. a better-informed one is to find a template, which will save you much time on debuging the css style. pick one from [here](http://jekyllthemes.org/).

The last work is to replace your private information in some  pages according to your favorites. the most important one is the _config.yml.

reminder: Do not forget to upload your last change into github. 
create your own repository on github and clone it on your work space:

```
git clone https://github.com/yourrepository 
```

push your change into github after change

```
git add .
git commit -m "some comments for this commit"
git push origin/master
```

### Run web server 

I'd introduce more detailed on this section... all of the below work is done on GCP instance

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

the main steps is all above, some things may help you to avoid trouble. 
1. set the the JEKYLL_ENV=production variable into your bash profile. if not you will find the navigator can not find the css files and image files
2. if you want to run the server as a background process and forward the output to one log file, please replace the step 4 as:

```
nohup bundle exec jekyll serve -H 0.0.0.0 -P 80 &> /var/log/mywebsit.log &2>1 &
```

anyother error can be fixed by google.

### Create a domain and reference to GCP server

I used [GoDaddy](https://www.godaddy.com/) to register my domain, it will take you some money. that work can be finished easily, the only difficault thing is to find a favourite name without be taken by other.

Change 'points to' the A field into your GCP public ip following [here](https://www.godaddy.com/help/change-my-ip-address-20134).

## Summarize

That's all my jobs of site building, to build a your own blog site is not difficult as you think. the most difficult thing is modifying the web content. rest thing is only labor.
