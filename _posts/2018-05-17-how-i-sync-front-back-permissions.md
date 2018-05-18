---
layout: post
title:  How I sync front-end and back-end permissions in single page app
date:   2018-05-16 21:51:33 -0600
categories: tip
tags: antd javascript react
---
Permissions are hard. Ensuring the javascript front-end of your application is working with the same set of permissions 
for a given entity as the backend is even harder. The first step towards madness is trying to code the business rules 
that grant/remove permissions in the frontend. This is opinion not fact. Some people may have managed to do this 
successfully but I've never met them. I like to manage permissions as follows:

### Step 1 - Enumerate the permissions
For each entity type in your system define an Enum that encapsulates all the possible permissions. Namespace your 
permissions. Choose good names.

{% highlight python %}{% raw %}
class EntityPermissionsEnum:
    EDIT = 'entity:edit'
    PUBLISH = 'entity:publish'
    VIEW = 'entity:view'
{% endraw %}{% endhighlight %}

### Step 2 - Centralize the permissions business logic
For each entity type in your system define a Permissions Class that has a _get_permissions(entity)_ class method on it

{% highlight python %}{% raw %}
class EntityPermissions:

    @classmethod
    def get_permissions(cls, requesting_user, entity):
        """
        Calculates the maximal set of permissions the requesting user has on the specified entity
        :param user requesting_user: the user in context
        :param model entity: the entity to get permissions on
        :return set: 
        """
        
        # Start with maximal permissions defined as a Set
        permissions = {
            EntityPermissionsEnum.VIEW,
            EntityPermissionsEnum.EDIT,
            EntityPermissionsEnum.PUBLISH,
        }
        
        # Discard permissions according to your business logic needs
        if entity.owner != requesting_user:
            permissions.discard(EntityPermissionsEnum.EDIT):
            
        # Use fancy set operations to keep code clean
        if not requesting_user.is_admin:
            permissions.intersection_update({EntityPermissionsEnum.VIEW, EntityPermissionsEnum.VOTE})
            
        return permissions
        
{% endraw %}{% endhighlight %}

### Step 3 - Check permissions on server-side using set membership
Checking for permissions is now a case of calling the _get_permissions()_ method and testing for the required permission
or permissions within the permissions set.

{% highlight python %}{% raw %}

permissions = EntityPermissions.get_permissions(requesting_user=requesting_user, entity=entity)
if EntityPermissionsEnum.VIEW not in permissions:
    raise PermissionsError()

{% endraw %}{% endhighlight %}

### Step 4 - Serialize the permissions along with the rest of entity on the way out
Wherever you serialize the entity in REST responses, add a generated field called __permissions__ that contains the list
of permissions. If you are using Django REST Framework you can use method serializer fields to dynamically call 
_get_permissions()_ and then mix this into a base serializer class.

{% highlight python %}{% raw %}
def rest_view():
    # get the entity...
    entity_dict = dict(entity)
    permissions = EntityPermissions.get_permissions(requesting_user=requesting_user, entity=entity)
    entity_dict['__permissions__'] = list(permissions)
    return json.dumps(entity_dict) 
{% endraw %}{% endhighlight %}

The serialized entity will now look like:
{% highlight json %}{% raw %}
{
    id: 1,
    title: 'blah',
    owner: 4,
    __permissions__: ['entity:view', 'entity:edit', 'entity:publish']
}
{% endraw %}{% endhighlight %}

### Step 5 - Use javascript Array methods to check permissions in frontend 
Use the new Array methods in ES6 to check permissions. We could use Maps, but they don't serialize to/from json nicely
so I find Arrays work best.
{% highlight javascript %}{% raw %}

const ENTITY_PERMISSIONS = {
    VIEW: 'entity:view',
    EDIT: 'entity:edit',
    PUBLISH: 'entity:publish',
}

if(entity.__permissions__.includes(ENTITY_PERMISSIONS.EDIT)){
    // Render Edit Button
}

{% endraw %}{% endhighlight %}

### Step 6 - Improvements

 * Automatically generate the javascript permissions constants from the enum classes to keep them in sync.
 * Cache the entities on the server for the lifetime of the request if they are expensive to generate
 

