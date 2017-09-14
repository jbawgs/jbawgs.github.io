---
layout: post
title: "Salesforce Rich Text Editor"
date: '2017-09-14 8:16:00 -0400'
comments: true
---
## Salesforce Rich Text Editor
### How to unlock the power of CKEditor
##### [`Gist Here`](https://gist.github.com/jbawgs/d2a97e71b299314b2a2f089a84fc0afc)

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

I just ripped this directly from the Salesforce page source, I honestly have no idea what most of these do.

When describing your new editor config in JS, you'll need to include these plugins in your Toolbar definition:
{%highlight js%}
{
     name: 'insert',
     items: ['sfdcImage', 'Table', 'CodeSnippet']
}
{%endhighlight%}

And you also need to add the configuration option `sfdcLabels`:
{%highlight js%}
sfdcLabels: {
    CkeMediaEmbed: {
        iframeMissing: 'Invalid &lt;iframe&gt; element. Please use valid code from the approved sites.',
        description: 'Use &lt;iframe&gt; code from DailyMotion, Vimeo, and Youtube.',
        title: 'Embed Multimedia Content',
        exampleTitle: 'Example:',
        subtitle: 'Paste &amp;lt;iframe&amp;gt; code here:',
        example: '\n            \n                &lt;iframe width=\&quot;560\&quot; height=\&quot;315\&quot; src=\&quot;https://www.youtube.com/embed/KcOm0TNvKBA\&quot; frameborder=\&quot;0\&quot; allowfullscreen&gt;&lt;/iframe&gt;\n            \n        '
    },
    CkeImagePaste: {
        CkeImagePasteWarning: 'Pasting an image is not working properly with Firefox, please use [Copy Image location] instead.'
    },
    CkeImageDialog: {
        infoTab_desc_info: 'Enter a description of the image for visually impaired users',
        uploadTab_desc: 'Description',
        defaultImageDescription: 'User-added image',
        uploadTab_file_info: 'Maximum size 1 MB. Only png, gif or jpeg',
        uploadTab_desc_info: 'Enter a description of the image for visually impaired users',
        imageUploadLimit_info: 'Max number of upload images exceeded',
        btn_insert_tooltip: 'Insert Image',
        httpUrlWarning: 'Are you sure you want to use an HTTP URL? Using HTTP image URLs may result in security warnings about insecure content. To avoid these warnings, use HTTPS image URLs instead.',
        title: 'Insert Image',
        error: 'Error:',
        uploadTab: 'Upload Image',
        wrongFileTypeError: 'You can insert only .gif .jpeg and .png files.',
        infoTab_url: 'URL',
        infoTab: 'Web Address',
        infoTab_url_info: 'Example: http://www.mysite.com/myimage.jpg',
        missingUrlError: 'You must enter a URL',
        uploadTab_file: 'Select Image',
        btn_update_tooltip: 'Update Image',
        infoTab_desc: 'Description',
        btn_insert: 'Insert',
        btn_update: 'Update',
        btn_upadte: 'Update',
        invalidUrlError: 'You can only use http:, https:, data:, //, /, or relative URL schemes.'
    },
    sfdcSwitchToText: {
        sfdcSwitchToTextAlt: 'Use plain text'
    }
}
{%endhighlight%}

This is also ripped from SF's editor config, and sets some strings needed for the plugins to work correctly. Sadly, I couldn't get the plugin sfdcMediaEmbed to work properly. It looks great in the editor, but when you save the text to the database it strips out any iframe tags. Looking at the source, it seems that this plugin is for use with knowledge articles. It took quite a bit of digging around through SF's source to get the editor to this level, and while I'd like to be able to post youtube embeds, what I have right now is a certain improvement. We now have formatting and styling options, code formatting, tables, blockquotes, and the glorious Maximize button for fullscreen editing.

A big improvement.

I'll conclude with a JS snippet that can be dropped into any Visualforce page that has a rich text input field. Just wrap it in &lt;script&gt; tags and paste it in. It isn't super graceful, but to get the SF plugins to work correctly I needed to pull a few config variables out of the default configuration, i.e. the `filebrowserImageUploadUrl` value, since the image upload plugin needs a security token I couldn't find access to anywhere else. This script checks every 50 milliseconds if the editor has been loaded, and when it has, steals the info it needs, kills the old one, and launches the shiny new one. its pretty seamless. So here's the sauce:
{%highlight js%}
//values we need from the old editor
var instanceName;
var uploadURL;
var extraPlugins;

//the default editor exists in our scope as the variable 'editor'
//get the name of the text area element the editor lives in
instanceName = editor.name;

//add an event that launches our code when the editor instance is loaded
editor.on('instanceReady', reInitEditor());

function reInitEditor() {
    //test if the value we need is null
    if (editor.config.filebrowserImageUploadUrl) {
        //Assign the values we need for our new editor instance
        uploadUrl = editor.config.filebrowserImageUploadUrl;
        xtraPlugins = editor.config.extraPlugins;

        //DESTROY !
        editor.destroy(true);

        //Init our new editor instance with config. Configuring the editor is well documented at http://docs.ckeditor.com/
        editor = CKEDITOR.replace(instanceName, {
            removePlugins: 'image',
            toolbar: [
                {
                    name: 'clipboard',
                    items: ['Undo', 'Redo']
                },
                {
                    name: 'styles',
                    items: ['Format', 'Font', 'FontSize']
                },
                {
                    name: 'basicstyles',
                    items: ['Bold', 'Italic', 'Underline', 'Strike', 'RemoveFormat', 'CopyFormatting']
                },
                {
                    name: 'colors',
                    items: ['TextColor', 'BGColor']
                },
                {
                    name: 'align',
                    items: ['JustifyLeft', 'JustifyCenter', 'JustifyRight', 'JustifyBlock']
                },
                {
                    name: 'links',
                    items: ['Link', 'Unlink']
                },
                {
                    name: 'paragraph',
                    items: ['NumberedList', 'BulletedList', '-', 'Outdent', 'Indent', '-', 'Blockquote']
                },
                {
                    name: 'insert',
                    items: ['sfdcImage', 'sfdcMediaEmbed', 'Table', 'CodeSnippet']
                },
                {
                    name: 'tools',
                    items: ['Maximize']
                },
                {
                    name: 'editing',
                    items: ['Scayt']
                },
                {
                    name: 'document',
                    items: ['Print', 'Source']
                }
            ],
            customConfig: '',
            extraAllowedContent: 'iframe',
            //xtraPlugins is a value we saved from the old instance to load the SF proprietary plugins
            extraPlugins: xtraPlugins + ',autoembed,embed,codesnippet',

            //This is a mess but necessary for SF plugins to function
            sfdcLabels: {
                CkeMediaEmbed: {
                    iframeMissing: 'Invalid &lt;iframe&gt; element. Please use valid code from the approved sites.',
                    description: 'Use &lt;iframe&gt; code from DailyMotion, Vimeo, and Youtube.',
                    title: 'Embed Multimedia Content',
                    exampleTitle: 'Example:',
                    subtitle: 'Paste &amp;lt;iframe&amp;gt; code here:',
                    example: '\n            \n                &lt;iframe width=\&quot;560\&quot; height=\&quot;315\&quot; src=\&quot;https://www.youtube.com/embed/KcOm0TNvKBA\&quot; frameborder=\&quot;0\&quot; allowfullscreen&gt;&lt;/iframe&gt;\n            \n        '
                },
                CkeImagePaste: {
                    CkeImagePasteWarning: 'Pasting an image is not working properly with Firefox, please use [Copy Image location] instead.'
                },
                CkeImageDialog: {
                    infoTab_desc_info: 'Enter a description of the image for visually impaired users',
                    uploadTab_desc: 'Description',
                    defaultImageDescription: 'User-added image',
                    uploadTab_file_info: 'Maximum size 1 MB. Only png, gif or jpeg',
                    uploadTab_desc_info: 'Enter a description of the image for visually impaired users',
                    imageUploadLimit_info: 'Max number of upload images exceeded',
                    btn_insert_tooltip: 'Insert Image',
                    httpUrlWarning: 'Are you sure you want to use an HTTP URL? Using HTTP image URLs may result in security warnings about insecure content. To avoid these warnings, use HTTPS image URLs instead.',
                    title: 'Insert Image',
                    error: 'Error:',
                    uploadTab: 'Upload Image',
                    wrongFileTypeError: 'You can insert only .gif .jpeg and .png files.',
                    infoTab_url: 'URL',
                    infoTab: 'Web Address',
                    infoTab_url_info: 'Example: http://www.mysite.com/myimage.jpg',
                    missingUrlError: 'You must enter a URL',
                    uploadTab_file: 'Select Image',
                    btn_update_tooltip: 'Update Image',
                    infoTab_desc: 'Description',
                    btn_insert: 'Insert',
                    btn_update: 'Update',
                    btn_upadte: 'Update',
                    invalidUrlError: 'You can only use http:, https:, data:, //, /, or relative URL schemes.'
                },
                sfdcSwitchToText: {
                    sfdcSwitchToTextAlt: 'Use plain text'
                }
            },

            //load up the URL and token from the old editor instance
            filebrowserImageUploadUrl: uploadUrl,
            codeSnippet_theme: 'monokai_sublime',
            contentsCss: [ 'https://cdn.ckeditor.com/4.6.1/standard-all/contents.css'],
            removeDialogTabs: 'image:advanced;',
            embed_provider: '//noembed.com/embed?url={url}&callback={callback}'     
        });
    } else {
        //Our IF test failed so we wait 50 millis
        setTimeout(reInitEditor, 50);
    }
}      
{%endhighlight%}