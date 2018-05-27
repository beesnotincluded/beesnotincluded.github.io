---
layout: post
title:  "Where I put business logic in my django apps"
date:   2018-05-26 
categories: tip
tags: python django webdev business-logic 
---
### So many options...

There are various places you _could_ put business logic in your Django apps. You could do as suggested and put it in the
model classes. This is the "Fat Model" approach because you end up with large Model Class definitions. You could put the
logic in the view or REST ViewSet (if you are using Django Rest Framework for example) but this is rarely a good idea
because it's hard to reuse across multiple views and leads to hard to read/maintain view code. You could put the logic
"somewhere else", perhaps in a custom layer that sits between your views and your models. I happen to favor putting 
business logic in the Model _Manager_ classes.

### The case for "Fat Model Managers"

Model Managers are a good candidate because
 * They can be referenced without an instance of a model, which is especially useful for __creating things__ 
 * Django's model create() method lives on the manager class and so too does the create_user() method for the auth 
 User Model - this seems like a clue that it might be a good place to put other important logic!
 * The Model Managers already have handy access to the model class and the queryset methods etc.
 * They are easily accessible to the rest of the app via: Model.objects 
 * Triggering side-effects from methods on Model Managers feels a lot less dirty than doing so from a Model method
 * This approach leaves you Model classes easy to read and simple, which makes working with your models easier and also
 makes it easier for other people looking at your code for the first time
 * You can have methods that manipulate lists of models at once. If you have, say a bulk update, you can do that within
 your model manager method, rather than having to loop over individual models to update each one in turn.
 * You can create multiple managers and attach them to a single ModelClass. This can be handy if you want to separate
 read from write actions or business logic from reporting logic etc.
 * Your model classes stay clean with no side-effects because you are just using the Django functionality and nothing more

### How I do this in practice
 
#### Step1: Define the QuerySet
 
{% highlight python %}{%- raw -%}
from django.db.models import QuerySet
 
class OrganizationQuerySet(QuerySet):
    """
    Custom Organization Query Set
    """

    # custom filter methods go here...
 
{% endraw %}{% endhighlight %}

#### Step 2: Define the manager


{% highlight python %}{%- raw -%}
from django.db.models import Manager

class OrganizationManager(Manager):

    def create_organization(self, requesting_user, parent, license_type, business_type=NA, hosting_platforms=NA,
                            org_units=NA, status=OrganizationStatusEnum.ACTIVE, **params):
        """
        Create a new org, triggering appropriate side effects and logs
        :return:
        """
        
        # You can reference the model class via self.model
        org = self.model(
            parent_id=parent_id,
            status=status,
        )
        
        # You can trigger side-effects, like logging events
        LogRecord.objects.log_create_organization(
            requesting_user=requesting_user,
            org=org
        )
        
        # Other business logic can go here
        
        # Return the mutated/created model instance for further processing
        return org
        
        
{% endraw %}{% endhighlight %}

#### Step 3: Associate the Manager and Queryset with the model class

{% highlight python %}{%- raw -%}
class Organization(Model):

    # fields go here ...

    objects = OrganizationManager.from_queryset(OrganizationQuerySet)()
    
{% endraw %}{% endhighlight %}

### Conventions
I always pass the requesting_user as the first argument to a model manager. This decouples the session logic from the 
manager logic and also allows the calling code to more easily perform actions as any particular user, especially for 
those tricky cases like password reset or registration. Even if you don't think you'll need  the requesting user at 
the moment, you'll probably need it later, so just pass it through to everything and you'll never have to refactor 
again.

I don't usually have read-only manager methods. I find that kind of business logic is best dealt with by custom filter
methods.

Try to always return the updated model instance from your manager methods. This can then be used by view code or by 
subsequent model managers to continue performing the necessary logic.


### Conclusion

Keep you models thin and put your business logic in your Managers. Create multiple Managers if they are getting too 
big and if they can be logically separated.


