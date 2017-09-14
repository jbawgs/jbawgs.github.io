---
layout: post
title: "Salesforce Rich Text Editor"
date: '2017-08-14 8:16:00 -0400'
comments: true
---
## Salesforce Rich Text Editor
### How to unlock the power of CKEditor

When editing Rich Text fields, Salesforce employs a library called CKEditor, a popular WYSIWYG text editor used in a variety of web applications. You can customize CKEditor to your liking when you embed it in a page, and Salesforce has really stripped down its functionality in many ways.

We can (mostly) fix that!

Lets compare the boring editor:
![Bo-ring]({{ site.url | append:site.baseurl }}/images/sadeditor.PNG)

To the awesome editor:
<!--more-->
![Just Right]({{ site.url | append:site.baseurl }}/images/happyeditor.PNG)

In this particular example, I was modifying the editor to facilitate writing blog posts for my Salesforce-as-a-backend blog project. The blog edit page is custom visualforce so we could get the javascript in there to reinitialize the editor. I haven't done this in Lightning - I may in the future, and I don't imagine the process will be vastly different.

I've added Font/Style/Size/Color/Text BG options, additional formatting (or strip formatting), blockquote, Source View for tweaking the post's source, and a fullscreen editing mode for distraction-free writing.

Documentation for CKEditor covers all of this and more in the [CKEditor Developer Guide](http://docs.ckeditor.com/#!/guide/dev_configuration)

When configuring the CKEditor used on Salesforce Rich Text fields, a little additional legwork is necessary, because Salesforce uses a couple of custom editor plugins to handle media uploads and such.

{%highlight js%}
extraPlugins = "sfdcImage,sfdcMediaEmbed,sfdcSmartLink,sfdcCodeBlock,sfdcTable,sfdcVfAjax4J";
{%endhighlight%}
