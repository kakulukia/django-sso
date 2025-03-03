# Django SSO (Single Sign-On) [Beta]

Implementation of Single Sign-On for Django.

Supported Django versions:

- `3.*`
- `4.*`

*It's recommended to force specify package versions while installing packages. This is for security and compatibility reasons.*

## How does it work?

You should install this library to your Django project, that you plan to use as the authorization gateway (this instance will keep all user accounts). Then add this library to services which you wanna authenticate via the SSO gateway, add a little bit of configuration and that's all!

## Installation

### Server side

1. Add the module named `django_sso.gateway` to the `INSTALLED_APPS` variable  

```python
# project/settings.py
INSTALLED_APPS = [
    # ...
    'django_sso.sso_gateway',
]
```

2. Migrate the gateway models

```python
./manage.py migrate sso_gateway
```

3. Add sso-urls to the project:

```python
# project/urls.py

urlpatterns = [
	# ...,
	path('', include('django_sso.sso_gateway.urls')),
]
```

4. In the admin panel you will see a new section named `SINGLE SIGN-ON`. And in `Subordinated services` you should create a new entry with:

- `Name` - Human name of the service
- `Base url` - URL for redirects and access to service endpoints from the server. (Like https://your-service.example).
- `Enabled` - Are subordinated services active. (Inactive services can’t communicate with the server and the server can’t communicate with it)
- `Token` - Automatically generated token you should paste into your services `settings.py`. Copy this for the next step of configuring the client.

5. Settings (if needed)

```python
# SSO settings section in the gateway side are optional
SSO = {
    # Timeout for the communication with subordinated services. (OPTIONAL)
    # This timeout is defined in seconds with a default value of 0.1s 
    # (100ms) per registered service.
    'SUBORDINATE_COMMUNICATION_TIMEOUT': 0.1,
    
    # Additional fields. (OPTIONAL). For more details look to part
    # named as "Send additional data to subordinated services"
    'ADDITIONAL_FIELDS': ('additiona_fields', 'from_user_model', 'and_related_models'),
}
```

Then server side is ready to use!

### Client side

When the library is attached to the client side project, the admin login form will be overridden with same view as `login/` at client side.

1) Add `django_sso.sso_service` to `INSTALLED_APPS` 

```python
# project/settings.py
INSTALLED_APPS = [
    # ...
    'django_sso.sso_service',
]
```

2) Add urls to service application

```python
# project/urls.py

urlpatterns = [
    # ...,
    path('', include('django_sso.sso_service.urls')),    
]
```

3) Setup settings variables

```python
# project/settings.py

# Django variable. URL for unlogged users. We redirect it to our view.
LOGIN_URL = '/login/'

SSO = {
    # Specify SSO server base url (REQUIRED)
    'ROOT': 'https://sso.project.test',
    
	# Specify application token obtained in the SSO server admin panel (REQUIRED)
	'TOKEN': 'reej8Vt5kbCPJM9mZQqYsvfxC...',
 	
    # Overriding event acceptor class (OPTIONAL). For more details read
    # "Overriding event acceptor in subordinated service" partition
    'EVENT_ACCEPTOR_CLASS': 'project.my_overrides.MySSOEventAcceptor'
}
```



## Default behavior summary (In case, when the SSO gateway and subordinated service implemented with the Django SSO)

### Behavior

When user created/updated on the gateway app - system will emit to all subbordinated services profile info (creates or updates profile data with full list of basic and additional fields.

When direct related model (provided in ADDITIONAL_FIELDS setting) created or changed - system will emit event to all services with information about changes. By default - additional fields updation - on developer hands (you must override EventAcceptor class and provide own actions for events. You can read about it below). When deleted - system will send nulls same as sends data to all new/old related users.

When the user deauthenticated the Django SSO package will emit deauthentication event and instantly purge the sessions on subbordinated services.

When user deleted on the gateway side - the gateway app will emit event and in subordinated services user instantly will disabled if provided **is_active** property. By Django authentication subystem rules - if user have **is_active=False** - system will not authenticate it.



### Recomendations and importants

In your project gateway and all subordinated services must have same major version *(MAJOR.MINOR.CHORE_OR_FIX)*. Also very recomendet to install package with fixed version. Autoupdates may brokes your because breaking changes.

You **must** prevent overriding all data, wich coming from the SSO gateway app. You can change additional fields of the your user model in subbordinated service, but fields, provided from SSO exchange must be overriden only by the SSO gateway events.



## Structure

#### Server side URLs

- `login/` - central login form (you can override template `django_sso/login.html`) 
- `logout/` - central logout view. Clear all sessions on all resources for user

Internal library urls (endpoints for services):

- `sso/obtain/` - obtain <u>authorization request</u>
- `sso/get/` - get SSO token information. (Is authorized for this token? Get user identity from token. etc..)
- `sso/make_used/` - after successful authentication on client side need to mark authorization request as used.
- `sso/deauthenticate/` - services sends deauthentication requests to SSO-server. SSO server broadcasts all services to deauthenticate user
- `sso/debug/update_event/` - View for debugging the `SSO[ADDITIONAL_FIELDS]` setting. This URL enabled only in the Django `DEBUG` mode only. Here you can see emitted variables to all subordinated services per every login or affected model updation/deletion/creation.
- `__welcome/` - Sample view for testing. For logged and unlogged users.



#### Client side urls

- `login/` - login form. Intermediate form. Obtains authentication request and redirects to SSO server `/login`. 
- `logout/` - Deauthenticates the user and emits deauthentication event to SSO-server (to `sso/deauthenticate/` at server side).
- `sso/test/` - Page for testing the SSO mechanism immediately after installation of `django_sso`. When you open it in browser: If user are logged in - shows the username or redirects to SSO server and comes back after successful authentication.

Library urls for internal usage (endpoints for SSO-server side)

- `sso/event/` - Event acceptor from SSO server. Look at “**SSO with non-Django project / Accepting events**” section

- `sso/accept/` - User after successful authentication comes back. SSO-server redirect it to this URL for make Django authorization. Then after session is up - browser will redirect to the next URL.



## Overriding

### User and session storage

This library based on Django user subsystem and Django session subsystem. Also supports custom classes, but he must be based on classical Django classes (AbstractUser / AbstractBaseUser, etc..). This means that you have two ways. One: Do nothing, just install library to gateway/services and just use it. Two: Just pick own classes of user/session with based Django`s mechanisms.



### Overriding event acceptor in subordinated service

For event processing you must declare own class and inherit it from base class located in `django_sso.sso_service.backends.EventAcceptor`. Inheritance are necessary. Arguments must  absolutely matches for overridden methods. 

```python
# project/my_overrides.py
from django_sso.sso_service.backends import EventAcceptor, acceptor

# In case, when you need to do something after deauthentication
class MyEventAcceptor(EventAcceptor):
    @acceptor # Every event accpetor method must be decorated with it
    def deauthenticate(self, username):
        # Here you can do own actions before deauthentication
        super().deauthenticate(username)
        # And here you can do own actions after deauthentication

        
# In other case, when you need to override default behavior of class
class MyHardEventAcceptor(EventAcceptor):
    @acceptor
    def deauthenticate(self, username):
        # Here you do own actions
```

Method names are the same that event types. See [here](#accepting-events--additional-fields).

Then next put the path to this class into `settings.py`:

```python
SSO = {
    # Your settings
    ...

	'EVENT_ACCEPTOR_CLASS': 'project.my_overrides.MySSOEventAcceptor'
}
```



![](assets/sso_gateway_events_en.png)

Methods of the **EventAcceptor** class are the same that event names provided in scheme.



## Send additional data to subordinated services

By default the SSO gateway sends to subordinated services next fields (if provided in user model):
Also, this names are reserved and overriding of it is restricted in the ADDITIONAL_FIELDS setting.

- `is_staff` - Are user is staff member (can access admin panel).
- `is_active` - Obviously, activity status.
- `is_superuser` - Are user have full privileges.
- `user_identy` - User identity. Obtained from **YourUserModel.USERNAME_FIELD**. Unique and not null ever. E-Mail / Login etc.

Additional fields are not included in the default behavior for security reasons and for clarity of the project code and the code behavior.

But possible cases when you need to send more data, than only this flags. For this case the library have next way: 

<a name="imagine">Imagine, that we have next:</a>

```python
# Anything additional data
class Category(models.Model):
    name = models.CharField(max_length=12)

    def to_sso_representation(self):
        """
        This method will call if this instance in the ADDITIONAL_FIELDS provided just as a variable
        """
        return f'to_sso_representation() for Category #{self.name}'

    @property
    def my_var(self):
        return f'my_var for user #{self.id}'

    def my_method(self):
        """
        Your method that returns data (should be too fast)
        - it is recommended to mark this method for security reasons, as it is used in the SSO mechanism
        - recommended to be named like "sso_field__my_id_is" for code style and obviousness
        """
        return f'my_method() for user #{self.id}'


# One2One reverse related model wich extends user model
class ReversedAdvancedUserData(models.Model):
    something_else = CharField(max_length=12, default=None, null=True)


# Custom user model or the default auth.User model (no difference)
class UserModel(AbstractUser):
    my_custom_field = models.CharField(max_length=16, default=None, null=True)
    category = models.ForeignKey(Category, default=None, null=True, on_delete=models.CASCADE)
    advanced_data2 = models.OneToOneField(ReversedAdvancedUserData, null=True, on_delete=models.SET_NULL, related_name='data2')


# The One2One related model which extends user model (possible that you create this instance via signals)
class AdvancedUserData(models.Model):
    user = models.OneToOneField(UserModel, on_delete=models.CASCADE, related_name='data')
    my_data_field = models.IntegerField(default=1)
```



1. On the gateway side in `settings.py` add variable with implementation like next: 

```python
SSO = {
    # Additional fields wich send with update user event (Any fields related to user model)
    # This setting are optional
    'ADDITIONAL_FIELDS': (
        # It's a UserModel.my_custom_field
        'my_custom_field',

        # It's a Category.name from UserModel.category
        'category.name',

        # Will put Category.to_sso_representation (or __str__) by UserModel.category object
        'category',

        # If prop isset - will put AdvancedUserData.my_data_field
        'data.my_data_field',

        # Will get value from the AdvancedUserData.to_sso_representation method
        # or AdvancedUserData.__str__
        'data',

        # Will get value from ReversedAdvancedUserData.to_sso_representation method
        # or ReversedAdvancedUserData.__str__
        'advanced_data2',

        # This field is UserModel.my_custom_field with an alias named "an_alias"
        'my_custom_field:an_alias',

        # Also you can put properties and methods to sending data
        'category.my_method',  # Category.my_method will called
        'category.my_var',  # Get the Category.my_var prop value
    )
}
```

​	

- Value, that you put to fields, must be `bool` or `str` or `int` or `float` or `None`. If expected any other type: the `django-sso` library will try to cast to `str` or **will log error to console**. 
- When you will set the `SSO_ADDITIONAL_FIELDS` variable - system will subscribe to all **pre_save** signals of provided models and will get all described values and will emit event with it and list of users, related to changes. Event emits when any model has changed/deleted. If model deleted - Django SSO will emit fields filled with null to all related users to deleted object.  
- All finaly translated (to aliases or not) field names must be distinct and not overrides the basic variables (is_staff, enabled, is_superuser) else you will catch exception on project start.
- Any field may be aliased
- If methods execution time greater than 200ms - you got a detailed warning in console.
- Property order flow (all properties are related to user model at begin):![](assets/additional_fields_flow.png)
- <u>Allowed fields only from direct relations. Nested relations restricted.</u>
- If `ForeignKey` or  `OneToOneField` relation is `None` - `None` will placed to result.
- If during method execution occurs exception - to field will placed `None` and to console wil printed detailed warning. 
- Also exists page for debugging fields (see the gateway urls for more details).
	![image-20230120230127179](assets/sso_debug_event.png)





## Debug & development notice

All exceptions in SSO mechanism will be logged in console.

Note: The gateway and the services have to run on distinct domains to not mess with each others cookies. For development on a single machine use the `hosts` file to create multiple domains like this:

```hosts
127.0.0.1	sso_gateway
127.0.0.1	my_sso_service_1
127.0.0.1	my_sso_service_2
```
Otherwise, after successful authentication at the SSO gateway the system can't send the event to it's subordinated service. For upmost three different services you can also make use of localhost, 127.0.0.1 and 0.0.0.0 as distinct domain names. :) 

If an error occurs after the authorization at the gateway, the user will be redirected to `/?sso_broken_token=true` at the subordinated service. 

In case the authorization was successful but an error is caught while emitting the event the user will be redirected to the gateway page with url `/__welcome/fallback=true`.

# SSO with non-Django project

### Basics

Any external service must be registered in SSO server in admin panel section named  `SINGLE SIGN-ON / Subordinated services`. Then obtained token put to your script for next calls. And make service available directly for SSO server.

In next examples i’ll use sso_server.test meaning SSO server.

### Login page

When unlogged user visits login page, the backends need to request the SSO token from the SSO server.

Fields:

`token` - obtained from first step at SSO server

`next_url` - relative URL for redirect after successful authentication. (SSO will generate `Basic URL + Next URL` string and will forward user to it)

```bash
curl --request POST \
  --url http://sso_server.test/sso/obtain/ \
  --header 'Content-Type: application/x-www-form-urlencoded' \
  --header 'X-Token: IJj42agKd231SzinVYqJMqq0buinM0wU' \
  --data token=AhTu1Un5zef3eRMGsL3y7AbDt2213123123f \
  --data next_url=/successful/page/
```


If all succeeds, server will send you an authentication request token in JSON format.

```json
{
	"token": "NmyWRItAye0gDxX7CZhOFs2HKZtT3xyfdrq14TU"
}
```

Then

1) Write token to session. (In PHP - $_SESSION.)

2) Redirect user to http://sso_server.test/login/?sso=NmyWRItAye0gDxX7CZhOFs2HKZtT3xyfdrq14TU. You should put the obtained token to URL into “sso” parameter. User will be redirected to SSO login page. 

On SSO login page next:

If user successful logged on SSO, SSO sends to your event endpoint basic information about user. You should be it write to your authentication system. Then SSO server redirects user back to http://your_service.test/sso/accept/ where script recover SSO token from session and request information from SSO:

`token` - Service token.

`authentication_token` - SSO token, obtained in last step.

```bash
curl --request POST \
  --url http://sso_server.test/sso/get/ \
  --header 'Content-Type: application/x-www-form-urlencoded' \
  --header 'X-Token: IJj42agKdp31SzinVYqJMqq0buinM0wU' \
  --data token=AhTu1Un5zef3eRMGsL3y7AbDt2213123123f \
  --data authentication_token=NmyWRItAye0gDxX7CZhOFs2HKZtT3xyfdrq14TU
```

If user already authorized. It will be returned next JSON:

```json
{
    "authenticated": True, // Are successful authenticated
    "user_identy": "somebody", // User identy (login or email...)
    "next_url": "/admin/" // URL for redirect after successful auth
}
```

In any other case:

```json
{
	"error": "Authentication request token doesn't exist"
}
```

If all success. You need to notify SSO server that token is used. Do next:

`token` - Service token.

`authentication_token` - SSO token, obtained in last step.

```bash
curl --request POST \
  --url http://sso_server.test/sso/make_used/ \
  --header 'Content-Type: application/x-www-form-urlencoded' \
  --header 'X-Token: IJj42agKdp31SzinVYqJMqq0buinM0wU' \
  --data token=AhTu1Un5zef3eRMGsL3y7AbDt2213123123f \
  --data authentication_token=NmyWRItAye0gDxX7CZhOFs2HKZtT3xyfdrq14TU
```

And you will be get reply:

```json
{
	"ok": true
}
```



### Logout page

You first purge data from session. Then send to SSO server deauthentication event.

`token` - Service token.

`user_identy` - username or email field. Same, that you obtained from http://sso_server.test/sso/get/ at login procedure.

```bash
curl --request POST \
  --url http://127.0.0.1:5000/sso/deauthenticate/ \
  --header 'Content-Type: application/x-www-form-urlencoded' \
  --header 'X-Token: IJj42agKdp31SzinVYqJMqq0buinM0wU' \
  --data token=AhTu1Un5zef3eRMGsL3y7AbDt2213123123f \
  --data user_identy=someody
```

Will respond with

```json
{
    "ok": true
}
```

Meaning, that deauthentication completed for all services.



### Accepting events

You should create `/sso/event/` endpoint in your project.

When user does successful authentication on SSO server, when user deleted or changed on SSO server, this library will send events to all subordinated services information about it. Account was deleted or marked as superuser or something other - SSO-server will emit events to all subordinated services. 

For example. When user is marked as superuser, SSO-server will cast event to all subordinated services to `sso/event/`. Next written all possible events. `type`, `token` fields in event data are permanent.

Events sends in JSON format via POST request.



##### Create/update account

When user is created, updated, disabled, (un)marked as superuser and with every login. By `django-sso` behavior: Event fields `is_active`, `is_staff`, `is_superuser` will cast to `bool`.

```json
{
    "type": "update_account", // Event name
    "token": "kaIrVNHF4msyLBJeaD4hSO", // Service token to authenticate SSO server
    
    // Next fields may be not included in event. Because user model on SSO don't have it
    "fields": [
        "user_identy": "somebody", // Value from username field
        "is_active": True,  // Active user at SSO server
        "is_staff": True,  // User is staff member
        "is_superuser": True,  // User is superuser
        
        "custom_field": "Somethig from field", // Custom field 
        "data.field": 1, // Custom field from OneToOne related object or ForeginKey field if isset,
    ]
}
```



##### Deauthenticate

When user requested for deauthentication at any service or user deleted on the SSO gateway. This event will be emitted to all active subordinated services.

```json
{
    "type": "deauthenticate", // Event name
    "token": "kaIrVNHF4msyLBJeaD4hSO", // Service token to authenticate SSO server
    
    "user_identy": "somebody" // Value from username field
}
```



##### Delete account

When user has been deleted on the SSO gateway - system will emit event about it with next body:

```json
{
    "type": "delete", // Event name
    "token": "kaIrVNHF4msyLBJeaD4hSO", // Service token to authenticate SSO server
    
    "user_identy": "somebody" // value from username field
}
```



##### Change user identy

When user has been renamed on SSO gateway - system will emit event with old and new identy to rename user on all subordinated services.

```json
{
    "type": "change_user_identy", // Event name
    "token": "kaIrVNHF4msyLBJeaD4hSO", // Service token to authenticate SSO server
    
    "old": "old_user@email_or.login", // Old identy
    "new": "new_user@email_or.login" // New identy
}
```



##### <a name="accepting-events--additional-fields">Additional fields update</a>

If you set an _additional fields_ in the settings (`SSO['ADDITIONAL_FIELDS']`) and the fields are taken from models other than your `UserModel`: The SSO Gateway will generate the next event for each model separately and cast it to all subordinated services:

(following examples will be written in context of [imagintaion](#imagine))

```python
// In case when created or deleted

// For Category model will next:
{
	"type": "update_fields",
    "token": "kaIrVNHF4msyLBJeaD4hSO",
    
    "fields": [
        "category.name": "New name",
        "category.id": "New category id"
    ],
    
    "user_identities": [
        "admin",
        "test",
        "DAVIDhaker"
    ]
}

// For AdvancedUserData next:
{
    "type": "update_fields",
    "token": "kaIrVNHF4msyLBJeaD4hSO",
    
    "fields": [
        "data.my_data_field": "New value 2"
    ],
    
    "user_identities": [
        "admin",
        "test",
        "DAVIDhaker"
    ]
}

// If related object deleted, in "fields" all values will be null.
{
    "type": "fields_update",
    "token": "kaIrVNHF4msyLBJeaD4hSO",
    
    "fields": [
        "data.my_data_field": null
    ],
    
    "user_identities": [
        "admin",
        "test",
        "DAVIDhaker"
    ]
}
```



For all requests to `sso/event/` subordinated service must be return next reponces

```json
// In successful case
{
    "ok": True
}

// Else if failed
{
    "error": "Error description here"
}
```



# To do and coming fixes | roadmap

- [ ] Automatic test on all Django versions while push to master branch
- [x] Access control to subordinated services. Possibility to set available services per user.
	*Partially realized via the additional fields.*
- [ ] Event queue for pushing events instead of immediately pushing. For stability and efficiency and **for preventing of lost info, when subordinated service are off-line**.
- [ ] Integration with popular frameworks and making plug-ins for popular languages. (I can accept your code as part of project - link to repository, for example.)
- [ ] Integrate with Sentry
- [ ] Multilanguage docs
- [ ] Library provided mechanism for management user permissions.



# Support

There are more plans to make it better... Also i wanna to translate it to popular languages.

If you wanna to support project. You can do it via

Ethereum: 0x2BD7aA911861029feB08430EEB9a36DC9a8A14d2 (also accept any token :-) )

BUSD/BNB or any token (**BEP20**):  0x74e47ae3A26b8C5cD84d181595cC62723A1B114E

or any other way. You can mail ask about it.



Any thoughts: me@davidhaker.ru

With love for open source!
