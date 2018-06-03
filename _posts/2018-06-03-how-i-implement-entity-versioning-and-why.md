---
layout: post
title:  "How I implement entity level version control and why"
date:   2018-06-03 
categories: tip
tags: python django patterns opinion versioning version control
---

### We don't need versioning...oh wait, we do need versioning

If you are not sure if you need version control for a specific entity type in your system then you probably need 
version control. If not now, at some inconvenient point in the near future. But you can't justify the cost of 
implementing version control upfront, because you might _not_ need it and that wouldn't be very agile. So what should 
you do? I've been through this loop a couple of times and arrived at a fairly straightforward solution that you can roll 
out as and when you need it. It's based on the following assumptions.

1. The content being versioned is mostly text fields
1. You don't need multiple concurrent drafts
1. Version control is annoying to implement and systems without it are easier to develop, report, debug, build 
interfaces for etc.
1. Building version control up-front slows down development and leads to more re-work because you are trying to 
implement the basic workflow and version integration at the same time. Building and verifying the non-versioned 
system first is preferable to going 'all-in'.
1. New versions of entities are created less frequently than they are accessed (more reads than writes)
1. Most of the time you don't care about anything but the current version
1. You don't want other APIs to know about the existence of other versions
1. You need to render a history list of previous versions 
1. Other entities in the system will reference (via foreign keys) the entity in question


### Step 1. Don't implement versioning
Build your entity API and storage without caring about version. Focus on getting the behaviour, workflow and feature set 
figured out before you worry about version control

__Example:__ The Document Entity

We shall create a Model for the Document entity without version control


{% highlight python %}{%- raw -%}
class Document(Model):

    # fields go here ...
    name = models.CharField()
    summary = models.CharField(...)
    body = models.CharField(...)
    
    owner = models.ForeignKey(...)
    
{% endraw %}{% endhighlight %}

Now confirm that this meets the business requirements! It's a lot easier to refactor things before adding versioning

### Step 2. Adding the Version table

Create a Version table which shadows the fields that need to be versioned. Do not copy all the attributes. Just the ones 
that you actually want to version.

{% highlight python %}{%- raw -%}
class DocumentVersion(Model):

    # The doc that this DocumentVersion is associated with 
    document = models.ForeignKey(Document, on_delete=models.CASCADE, null=False, related_name='versions')

    # If true this is the current editable version of the policy
    current = models.BooleanField(null=False, default=True, db_index=True)
    
    # Version number assigned to this content version
    version_num = models.IntegerField(null=False)

    # versioned fields go here ...
    name = models.CharField(...)
    summary = models.CharField(...)
    body = models.CharField(...)
    
    # Note, we don't want to version the owner, since that is independent of the version control
    
{% endraw %}{% endhighlight %}

In addition, we add a _current_ boolean field that will be true only for the current editable version of the document.
You can set a database constraint on the combined document and current columns to ensure that only one 'current' version
exists for each document in the system.

We also need to add a version num to the Document Model class:

{% highlight python %}{%- raw -%}
class Document(Model):
    # ...
    version_num = models.IntegerField(null=False)
    
{% endraw %}{% endhighlight %}

### Step 3. How it works.

To __create a new document__, we just add new Document model setting the fields to the initial values we desire and setting 
the _version_num_ to 1. We do not need to create a version model yet.


    +--------------------------+
    | Document Model           |
    +--------------------------+
    | id          | 1          |
    +--------------------------+
    | name        | Doc1       |
    +--------------------------+
    | summary     | A doc...   |
    +--------------------------+
    | body        | Lorem Ip...|
    +--------------------------+
    | version_num | 1          |
    +--------------------------+
    
To __create a new version__, we create a new draft version by creating a DocumentVersion model, copying the following 
field data: name, summary, body. We set the version_num to one more than the version_num on the Document model. And we 
set the current flag to true.

    +--------------------------+
    | DocumentVersion  Model   |
    +--------------------------+
    | id          | 1          |
    +--------------------------+
    | name        | Doc1       |
    +--------------------------+
    | summary     | A doc...   |
    +--------------------------+
    | body        | Lorem Ip...|
    +--------------------------+
    | version_num | 2          |
    +--------------------------+
    | current     | true       |
    +--------------------------+

To __edit the draft version__, we simple allow the user to directly manipulate the name, summary and body fields on the
model we just created. It is always possible for the system to locate the current draft for a given doc id by simply
searching for the row in the version table with matching doc ID and current == True. You can even code this up as a
helper method on the Document Model class (assuming django).


    +--------------------------+
    | DocumentVersion  Model   |
    +--------------------------+
    | id          | 1          |
    +--------------------------+
    | name        | EDIT       |
    +--------------------------+
    | summary     | Edited doc.|
    +--------------------------+
    | body        | Edit Ip... |
    +--------------------------+
    | version_num | 2          |
    +--------------------------+
    | current     | true       |
    +--------------------------+
    
To __publish the draft version__ you just have to straight-copy the following fields from the DocumentVersion Model: 
name, summary, body and version_num to the associated Document Model. In the same transaction you'll have to set the 
current status of the DocumentVersion to false:

    +--------------------------+    +--------------------------+
    | Document Model           |    | DocumentVersion  Model   |
    +--------------------------+    +--------------------------+
    | id          | 1          |    | id          | 1          |
    +--------------------------+    +--------------------------+
    | name        | EDIT       |<-- | name        | EDIT       |
    +--------------------------+    +--------------------------+
    | summary     | Edited doc |<-- | summary     | Edited doc.|
    +--------------------------+    +--------------------------+
    | body        | Edit Ip... |<-- | body        | Edit Ip... |
    +--------------------------+    +--------------------------+
    | version_num | 2          |<-- | version_num | 2          |
    +--------------------------+    +--------------------------+
                                    | current     | false      |
                                    +--------------------------+

To __delete__ a current draft you can simply delete the record from the DocumentVersion table. No other work is requried.

To __render a history view__ of the Document, you can just output the contents of the DocumentVersion table for rows
matching the document ID in question.

To __render the current published version__ you just simply render the fields from the Document Model 

To __render the current draft version__ you just simply render the fields from the current DocumentVersion Model. The field names and types
are the same so you should be able to interchange them fairly readily.

### Advantages

 * The original entity table (e.g. Document table) always contains the latest published content. There is no need for 
 other APIs or modules to know anything about versions or have to filter out drafts etc.
 * Other entities can maintain references to the entity without having to update them when new versions are published
 * Retrieving history is a simple filtered query, no joins required.
 * The EntityVersion table always contains the full history
 * Conceptually easy to follow and debug
 * Readily accommodates more complex edit workflows like approval steps  etc.
 * If necessary you could truncate the Version table without impacting the rest of the application
 * Very simple logic 
 * Easy to integrate 'after the fact'
 
### Disadvantages
 * Needs safeguards for concurrent editing
 * Cannot easily support multiple 'current' drafts (although this is rarely necessary)
    
    
    
    
    
    
    
    
    
    
    
    
    
