---
layout: post
title: "Blog Apex Controller"
comments: true
---
# Enabling Tags

The next feature I wanted to enable was the ability to tag posts simply, and have the controller/vf combo parse and separate these into links which one could click and get a list of posts with the same tag. This presented a challenge whose solution induced much eye rolling on my part. Here's how it works.

When you're creating a new post in your org, you have a field beneath the post body like so: ![List of tags like so: tag1, tag2, tag3]({{ site.url | append:site.baseurl }}/images/tag-field.jpeg){: .wrap-img-right }

And you simply type in your tags, and the rest is automagically performed by the hidden robot fairies.![Tags as links in VF Page]({{ site.url | append:site.baseurl }}/images/tag-impl.png)
In the controller we have a Map<String, List<String>> which stores a post's Salesforce ID as the key, and a List of strings (that post's tags) as the value. Its declared with get/set at the top scope so that we can get at it with the VF page, and assigned whenever the getPosts() method is run. Here's the pertinent block of code in the controller:
{% highlight java %}
    tags = new Map<String,List<String>>();
    for(Blog_Post__c post : posts){
          if(post.Tags__c == null) post.Tags__c = ',';
               String[] theTags = post.Tags__c.split(',');
                for(String tag : theTags){
                      tag.trim();
                }
          tags.put(post.Id, theTags);
    } 
{% endhighlight %}
Breaking it down:

1. Init the map;
1. Loop through the posts
1. If this post doesn't have any tags, put a comma in the Tags__c field as a janky workaround for the lack of null checking in Visualforce
1. Init an array of string, theTags, and use the split() instance method of String to split up the Tags__c field into a new String for each individual tag
1. Start a loop through the list of tag Strings we just created.
1. Use the trim() instance method of String to cut out any whitespace from the tag.
1. xD
1. Assign the ID of this post and its list of tags to the map.

The split and trim methods were super important here. We tell it how to split a long string like this.:
{% highlight java %}
String longString = 'Mary had a little lamb';
String[] shortStrings = longString.split(' '); //This causes the string to be split by SPACES.
{% endhighlight %}
Now we have a list of strings that looks like this:

`['Mary', 'had', 'a', 'little', 'lamb']`

For the blog, we've got a string that look like this:

`'apex, visualforce, bugs, nightmares'`

When we use String.split(',') //split the string by commas we get a list of strings that looks like this:

`['apex', ' visualforce', ' bugs', ' nightmares']` <<<<------ OMG

It never crossed my mind those extra spaces at the beginning of some of the tags would cause me the hours-long nightmare that it totally did. In fact it never crossed my mind that those spaces existed, which is why it took two hours to fix. The solution was String.trim(). The trim() method just removes any whitespace from the beginning and end of your string. Problem solved. ![hackerman]({{ site.url | append:site.baseurl }}/images/small-hackerman.jpeg)
SO! Now we have a map we can use, we just feed it a post ID and it spits out a list of the tags for that post.
We render these on the VF page with a `<apex:repeat>` block.
{% highlight java %}
{% raw %}
    <apex:repeat value="{!tags[post.id]}" var="tag">
         <apex:commandLink action="{!taggedPosts}" value="{!tag}">
              <apex:param assignTo="{!clickedTag}" value="{!tag}" name="clickedTag"/>
         </apex:commandLink> |
    </apex:repeat>
{% endraw %}
{% endhighlight %}

Lets break that down.

1. Start a repeat block. We are already inside a repeat block that runs through each post (which can be seen in a previous post), and this repeat runs through each tag for that post. We give the repeat block the value tags[post.id] where tags is the name of my map, and post is the variable that represents the post we're looking at right now. var="tag" assigns each string from our map to a variable called "tag" temporarily, much like in a for loop.
1. We create a commandLink with an action that runs a method in the controller called taggedPosts, and value="{!tag}" means that the current value of the tag variable will be what's displayed on the page.
1. `<apex:param>` is how we pass some data back to our controller. The controller has a variable called clickedTag and we're assigning it the value of our tag variable.
1. Close the commandLink block and add a nice little separator.
1. Repeat for each tag in the list.
When a tag gets clicked, the `<apex:param>` tag goes to work, and lets our controller know to do a new SOQL query like this:
{% highlight java %}
posts = [SELECT Id, Name, Body__c, Author__c, Sub_Heading__c, CreatedDate, LastModifiedDate, Tags__c, CreatedBy.Name
                                        FROM Blog_Post__c
                                        WHERE Published__c = TRUE AND Tags__c LIKE :clickedTag
                                        ORDER BY CreatedDate DESC
                                        LIMIT :list_size
                                        OFFSET :counter];
{% endhighlight %}
We can use the `LIKE` keyword in our query to search the `Tag__c` field of our posts for the clicked tag. This has some limitations. You can't use the `LIKE` keyword to search really long fields. 255 is the limit. If you want to search something larger, you need to use a SOSL search instead.

So that's how that works. Hopefully more helpful than confusing. I'll update the previous post about the VF page/Controller to the current version of the source if you want to see it in context. 