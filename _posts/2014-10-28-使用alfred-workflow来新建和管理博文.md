---
layout : post
categories: [其他]
tags : [github, alfred, blog]
keywords : github page, alfred workflow, blog
excerpt: 
---
{% include JB/setup %}


gp_manager
==========

github page manager, alfred workflow

##Feature

* new post file use default templet
* edit post file by search keyword
* edit templet

##How to install

1. download the last release zip: [gp_manager.alfredworkflow.zip](https://github.com/liang8305/gp_manager/releases/download/v1.0/gp_manager.alfredworkflow.zip)
1. unzip `gp_manager.alfredworkflow.zip` you will get `gp_manager.alfredworkflow`
1. double click the `gp_manager.alfredworkflow` file , alfred will open and install it



##How to use

alfred keyword is `gp`

![7da330f5-e6fb-4a48-835b-e7caf2bfb001]({{ site.image_dir }}/2015/20150513034730.png)

##Edit default templet:

	gp templet *search*

##New post file:

it will create new post file, filename format `{yyyy-mm-dd}-{title}.md`

  	gp new title
  	
it will create new post file in dir `{dirname}`

	gp new dirname/title
  	
![9c63d7ed-19e3-4cdc-b4b1-fa879d4b601d]({{ site.image_dir }}/2015/20150513034807.png)
  
##Edit post file

it will search the file

  	gp edit *search* 
  	
![70f992ae-3a6d-4e10-9230-074506cbbb0b]({{ site.image_dir }}/2015/20150513034819.png)

