---
layout: post
title: "Implementing a Blog in Visualforce"
date: '2017-05-31 12:58:57 -0400'
comments: true
---
# Chugging the Kool-Aid

In my Salesforce dev-org I've created a custom object, `Blog_Post__c`, which has the fields `Name`, `Body__c`, `Author__c`, `Sub_Heading__c` and `Tag__c`. The Body field is of type Rich Text field, and we use the built in Salesforce functionality to create, edit, and publish new blog posts. We have an Apex controller, which retrieves blog posts and handles requests for records and tag searches. It also contains methods for controlling pagination.

Peek the Apex: <!--more-->
{% highlight java %}
public with sharing class BlogIndexController {
    private Integer counter=0;
    private Integer list_size=5;
    public Integer total_size;
    public Map<String,List<String>> tags {get; set;}
    public string clickedTag {get; set;}
    public String pageID = ApexPages.currentPage().getParameters().get('pageId');
    
    public BlogIndexController() {
        total_size = [SELECT count() FROM Blog_Post__c];
        clickedTag = ApexPages.currentPage().getParameters().get('tag');
    }
    

    public Blog_Post__c[] getPosts(){
        Blog_Post__c[] posts;
        try{
            if(clickedTag == null){
                posts = [SELECT Id, Name, Body__c, Author__c, Sub_Heading__c, CreatedDate, LastModifiedDate, Tags__c, CreatedBy.Name
                        FROM Blog_Post__c 
                        WHERE Published__c = TRUE
                        ORDER BY CreatedDate DESC 
                        LIMIT :list_size 
                        OFFSET :counter];
            } else {
                String searchString = '%' + clickedTag + '%';
                posts = [SELECT Id, Name, Body__c, Author__c, Sub_Heading__c, CreatedDate, LastModifiedDate, Tags__c, CreatedBy.Name
                        FROM Blog_Post__c 
                        WHERE Published__c = TRUE AND Tags__c LIKE :searchString
                        ORDER BY CreatedDate DESC 
                        LIMIT :list_size 
                        OFFSET :counter];
            }
            if(pageID != null){
                posts = [SELECT Id, Name, Body__c, Author__c, Sub_Heading__c, CreatedDate, LastModifiedDate, Tags__c, CreatedBy.Name
                        FROM Blog_Post__c 
                        WHERE Id = :pageID];
            }
            
            tags = new Map<String,List<String>>();
            for(Blog_Post__c post : posts){
                if(post.Tags__c == null) post.Tags__c = ',';
                String[] theTags = post.Tags__c.split(',');
                for(String tag : theTags){
                    tag.trim();
                }
                tags.put(post.Id, theTags);
            } 
            return posts;
        } catch(QueryException e){
            ApexPages.addMessages(e);
            return null;
        }        
    }
    
    public PageReference taggedPosts(){
        pageID = null;
        System.debug('CLICKED - ' + clickedTag);
        return null;
    }
    
    public PageReference beginning(){
        counter = 0;
        return null;
    }
    
    public PageReference previous(){
        counter -= list_size;
        return null;
    }
    
    public PageReference next(){
        counter += list_size;
        return null;
    }
    
    public PageReference end(){
        counter = total_size - math.mod(total_size, list_size);
        return null;
    }
    
    public Boolean disablePrevious(){
        if(counter>0){
            return false;
        } else {
            return true;
        }
    }
    
    public Boolean disableNext(){
        if(counter + list_size < total_size){
            return false;
        } else {
            return true;
        }
    }
    
    public Integer getTotalSize(){
        return total_size;
    }
    
    public Integer getPageNumber(){
        return counter/list_size + 1;
    }
    
    public Integer getTotalPages(){
        if(math.mod(total_size, list_size) > 0){
            return (total_size/list_size +1);
        } else {
            return (total_size/list_size);
        }
    }
}
{% endhighlight %}

We use Salesforce Sites to surface a public Visualforce page at http://cappblog-developer-edition.na50.force.com/

The visualforce page accepts a list of posts from the controller and displays them in a branded page. Posts have clickable tags that return a new list of posts that contain said tag.

Here's the VF page:

{% highlight xml %}
{% raw %}
<apex:page controller="BlogIndexController" showHeader="false" sidebar="false" standardStylesheets="false">
<apex:messages ></apex:messages>
    <title>SF.BLOG</title>
    <apex:form >
    <div class="uk-container uk-dark">
            <img src="{!URLFOR($Resource.logo)}" alt=""/>
            
            <h3 class="uk-text-right"><span>{!IF(ISNULL(clickedTag), "", "Displaying posts tagged " + clickedTag + ".")}</span></h3>
    	<apex:repeat value="{!posts}" var="post">
            <h1 class="uk-heading-line"><span><a class="titleLinks" href="/Blog_Index?pageID={!post.id}">{!post.Name}</a></span></h1>
            <article class="uk-article">
                <h3 class="uk-article-title">{!post.Sub_Heading__c}</h3>
                <p class="uk-article-meta">Posted by {!post.CreatedBy.Name} on {!post.CreatedDate}.</p>
                <apex:outputText value="{!post.Body__c}" escape="false" />
                <p class="uk-article-meta" id="{!post.Id}">
                    <apex:repeat value="{!tags[post.id]}" var="tag">
                        <apex:commandLink action="{!taggedPosts}" value="| {!tag} " styleClass="titleLinks">
                            <apex:param assignTo="{!clickedTag}" value="{!tag}" name="clickedTag"/>
                        </apex:commandLink>
                    </apex:repeat>|
                </p>
            </article>
        </apex:repeat>
        
            <apex:panelGrid columns="2">
                <apex:commandLink action="{!previous}" styleclass="titleLinks">Previous</apex:commandlink>
                <apex:commandLink action="{!next}" styleclass="titleLinks">Next</apex:commandlink>
            </apex:panelGrid>
        
    </div>
    </apex:form>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/uikit/3.0.0-beta.22/css/uikit.min.css" />
    <script src="https://cdnjs.cloudflare.com/ajax/libs/uikit/3.0.0-beta.22/js/uikit.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/uikit/3.0.0-beta.22/js/uikit-icons.min.js"></script>
    
    <style>
        .uk-container{
            background-color:#191919;
        }
        a.titleLinks:link{color:#e21f3d;}
        a.titleLinks:visited{color:#e21f3d;}
        a.titleLinks:hover{color:#e01c3b;}
        a:link{color:#f0b0b0;}
        a:visited{color:#f0b0b0;}
        a:hover{color:#b0b0b0;}
        body{
            color:#d4d4d4;
            background-color: #1e1e1e;
        }
        h1{
            color:#e21f3d;
        }
        pre{
            outline: 0px solid transparent;
            padding: 0px 0px; 
            border: 0px double silver;
            background: #f00; 
            overflow: auto;
        }
    </style>
    <link rel="stylesheet" href="//cdnjs.cloudflare.com/ajax/libs/highlight.js/9.11.0/styles/monokai-sublime.min.css"/>
<script src="//cdnjs.cloudflare.com/ajax/libs/highlight.js/9.11.0/highlight.min.js"></script>
<script>hljs.initHighlightingOnLoad();</script>
</apex:page>
{% endraw %}
{% endhighlight %}