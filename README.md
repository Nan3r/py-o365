# Py-O365 - Office 365 API made easy

This project aims is to make it easy to interact with Office 365 Email, Contacts, Calendar, OneDrive, etc.

This project is based on the super work done by [Toben Archer](https://github.com/Narcolapser) [Python-O365](https://github.com/Narcolapser/python-o365).
The oauth part is based on the work done by [Royce Melborn](https://github.com/roycem90) which is now integrated with the original project.

I just want to make this project different in almost every sense, and make it also more pythonic (no getters and setters, etc.) and make it also compatible with oauth and basic auth.
So I ended up rewriting the hole project from scratch.

The result is a package that provides a lot of the Office 365 API capabilities.

This is for example how you send a message:

```python
from O365 import Account

credentials = ('username@example.com', 'my_password')

account = Account(credentials, auth_method='basic')
m = account.new_message()
m.to.add('to_example@example.com')
m.subject = 'Testing!'
m.body = "George Best quote: I've stopped drinking, but only while I'm asleep."
m.send()
```

**Python 3.4 is the minimum required**... I was very tempted to just go for 3.6 and use f-strings. Those are fantastic!

This project was also a learning resource for me. This is a list of not so common python characteristics used in this project:
- New unpacking technics: `def method(argument, *, with_name=None, **other_params):`
- Enums: `from enum import Enum`
- Factory paradigm
- Package organization
- Timezone conversion and timezone aware datetimes
- Etc. (see the code!)

> **This project is in early development.** Changes that can break your code may be commited. If you want to help please fell free to fork and make pull requests.

## Table of contents

- [Protocols](#protocols)
- [Authentication](#authentication)
- [Account Class and Modularity](#account)
- [MailBox](#mailbox)
- [AddressBook](#addressbook)
- [Calendar](#calendar)
- [OneDrive](#onedrive)
- [Utils](#utils)


## Protocols
Protocols handles the aspects of comunications between different APIs.
This project uses by default either the Office 365 APIs or Microsoft Graph APIs.
But, you can use many other Microsoft APIs as long as you implement the protocol needed.

You can use one or the other:

- `MSOffice365Protocol` to use the [Office 365 API](https://msdn.microsoft.com/en-us/office/office365/api/api-catalog)
- `MSGraphProtocol` to use the [Microsoft Graph API](https://developer.microsoft.com/en-us/graph/docs/concepts/overview)

To have a protocol that works with basic authentication a `BasicAuthProtocol` (that inherits from `MSOffice365Protocol`) is also provided for convenience.

Both protocols allow pretty much the same options (depending on the api version used).

The `Account` Class  will select the most apropriate protocol based on the auth method:
- When using basic authentication the protocol defaults to `BasicAuthProtocol` (`MSOffice365Protocol`) api version 1.0 on office365 endpoint (because Microsoft Graph doesn't allow basic authentication).
- When using oauth authentication the protocol defaults to `MSGraphProtocol`.

You can implement your own protocols by inheriting from `Protocol` to communicate with other Microsoft APIs.

You can instantiate protocols like this:
```python
from O365 import MSGraphProtocol

# try the api version beta of the Microsoft Graph endpoint.
protocol = MSGraphProtocol(api_version='beta')  # MSGraphProtocol defaults to v1.0 api version
```

##### Resources:
Each API endpoint requires a resource. This usually defines the owner of the data.
Every protocol defaults to resource 'ME'. 'ME' is the user which has given consent, but you can change this behaviour by providing a different default resource to the protocol constructor.

For example when accesing a shared mailbox:

```python
# ...
my_protocol = MSGraphProtocol(default_resource='shared_mailbox@example.com')

account = Account(credentials=my_credentials, protocol=my_protocol)

# now account is accesing the shared_mailbox@example.com in every api call.
shared_mailbox_messages = account.mailbox().get_messages()
```

Instead of defining the resource used at the protocol level, you can provide it per use case as follows:
```python
# ...
account = Account(credentials=my_credentials)  # account defaults to 'ME' resource

mailbox = account.mailbox('shared_mailbox@example.com')  # mailbox is using 'shared_mailbox@example.com' resource instead of 'ME'

# or:

message = Message(parent=account, main_resource='shared_mailbox@example.com')  # message is using 'shared_mailbox@example.com' resource
```


## Authentication
There are two types of authentication provided:

- Basic authentication: using just the username and password
- Oauth authentication: using an authentication token provided after user consent. This is the default authentication.

The `Connection` Class handles the authentication.

#### Basic Authentication
Just pass auth_method argument with 'basic' (or `AUTH_METHOD.BASIC` enum) parameter and provide the username and password as a tuple to the credentials argument of either the `Account` or the `Connection` class.
`Account` already creates a connection for you so you don't need to create a specific Connection object (See [Account Class and Modularity](#account)).
```python
from O365 import Account, AUTH_METHOD

credentials = ('username@example.com', 'my_password')

account = Account(credentials, auth_method=AUTH_METHOD.BASIC)
```

**Basic Authentication only works with Office 365 Api version v1.0 and until November 1 2018.**

#### Oauth Authentication
This is the recommended way of authenticating.
This section is explained using Microsoft Graph Protocol, almost the same applies to the Office 365 REST API.

##### Permissions and Scopes:
When using oauth you create an application and allow some resources to be accesed and used by it's users.
Then the user can request access to one or more of this resources by providing scopes to the oauth provider.

For example your application can have Calendar.Read, Mail.ReadWrite and Mail.Send permissions, but the application can request access only to the Mail.ReadWrite and Mail.Send permission.
This is done by providing scopes to the connection object like so:
```python
from O365 import Connection, AUTH_METHOD

credentials = ('client_id', 'client_secret')

scopes = ['https://graph.microsoft.com/Mail.ReadWrite', 'https://graph.microsoft.com/Mail.Send']

con = Connection(credentials, auth_method=AUTH_METHOD.OAUTH, scopes=scopes)
```

Scope implementation depends on the protocol used. So by using protocol data you can automatically set the scopes needed:

You can get the same scopes as before using protocols like this:

```python
protocol_graph = MSGraphProtocol()

scopes_graph = protocol.get_scopes_for('message all')
# scopes here are: ['https://graph.microsoft.com/Mail.ReadWrite', 'https://graph.microsoft.com/Mail.Send']

protocol_office = MSOffice365Protocol()

scopes_office = protocol.get_scopes_for('message all')
# scopes here are: ['https://outlook.office.com/Mail.ReadWrite', 'https://outlook.office.com/Mail.Send']

con = Connection(credentials, auth_method=AUTH_METHOD.OAUTH, scopes=scopes_graph)
```


##### Authentication Flow
1. To work with oauth you first need to register your application at [Microsoft Application Registration Portal](https://apps.dev.microsoft.com/).

    1. Login at [Microsoft Application Registration Portal](https://apps.dev.microsoft.com/)
    2. Create an app, note your app id (client_id)
    3. Generate a new password (client_secret) under "Application Secrets" section
    4. Under the "Platform" section, add a new Web platform and set "https://outlook.office365.com/owa/" as the redirect URL
    5. Under "Microsoft Graph Permissions" section, add the delegated permissions you want (see scopes), as an example, to read and send emails use:
        1. Mail.ReadWrite
        2. Mail.Send
        3. User.Read

2. Then you need to login for the first time to get the access token by consenting the application to access the resources it needs.
    1. First get the authorization url.
        ```python
        url = account.connection.get_authorization_url()
        ```
    2. The user must visit this url and give consent to the application. When consent is given, the page will rediret to: "https://outlook.office365.com/owa/".

       Then the user must copy the resulting page url and give it to the connection object:

        ```python

        result_url = input('Paste the result url here...')

        account.connection.request_token(result_url)  # This, if succesful, will store the token in a txt file on the user project folder.
        ```

        <span style="color:red">Take care, the access token must remain protected from unauthorized users.</span>

    3. At this point you will have an access token that will provide valid credentials when using the api. If you change the scope requested, then the current token won't work, and you will need the user to give consent again on the application to gain access to the new scopes requested.

    The access token only lasts 60 minutes, but the app will automatically request new tokens through the refresh tokens, but note that a refresh token only lasts for 90 days. So you must use it before or you will need to request a new access token again (no new consent needed by the user, just a login).

## Account Class and Modularity <a name="account"></a>
Usually you will only need to work with the `Account` Class. This is a wrapper around all functionality.

But you can also work only with the pieces you want.

For example, instead of:
```python
from O365 import Account

account = Account(('client_id', 'client_secret'), auth_method='oauth')
message = account.new_message()
# ...
mailbox = account.mailbox()
# ...
```

You can work only with the required pieces:

```python
from O365 import Connection, MSGraphProtocol, Message, MailBox,

my_protocol = MSGraphProtocol()
con = Connection(('client_id', 'client_secret'), auth_method='oauth')

message = Message(con=con, protocol=my_protocol)
# ...
mailbox = Mailbox(con=con, protocol=my_protocol)
message2 = Message(parent=mailbox)  # message will inherit the connection and protocol from mailbox when using parent.
# ...
```

It's also easy to implement a custom Class.

Just Inherit from ApiComponent, define the endpoints, and use the connection to make requests. If needed also inherit from Protocol to handle different comunications aspects with the API server.

```python
class CustomClass(ApiComponent):
    _endpoints = {'my_url_key': '/customendpoint'}
    
    def __init__(self, *, parent=None, con=None, **kwargs):
        super().__init__(parent=parent, con=con, **kwargs)
        # ...

    def do_some_stuff(self):
        
        # self.build_url just merges the protocol service_url with the enpoint passed as a parameter
        # to change the service_url implement your own protocol inherinting from Protocol Class
        url = self.build_url(self._endpoints.get('my_url_key'))  
        
        my_params = {'param1': 'param1'}

        response = self.con.get(url, params=my_params)  # note the use of the connection here.

        # handle response and return to the user...
```

## MailBox
Mailbox groups the funcionality of both the messages and the email folders.

```python
mailbox = account.mailbox()

inbox = mailbox.inbox_folder()

for message in inbox.get_messages():
    print(message)

sent_folder = mailbox.sent_folder()

for message in sent_folder.get_messages():
    print(message)

m = mailbox.new_message()

m.to.add('to_example@example.com')
m.body = 'George Best quote: In 1969 I gave up women and alcohol - it was the worst 20 minutes of my life.'
m.save_draft()
```

#### Email Folder
Represents a Folder within your email mailbox.

You can get any folder in your mailbox by requesting child folders or filtering by name.

```python
mailbox = account.mailbox()

archive = mailbox.get_folder(folder_name='archive')  # get a folder with 'archive' name

child_folders = archive.get_folders(25) # get at most 25 child folders of 'archive' folder

for folder in child_folders:
    print(folder.name, folder.parent_id)

archive.create_child_folder('George Best Quotes')
```

#### Message
An email object with all it's data and methods.

Creating a draft message is as easy as this:
```python
message = mailbox.new_message()
message.to.add(['example1@example.com', 'example2@example.com'])
message.sender.address = 'my_shared_account@example.com'  # changing the from address
message.body = 'George Best quote: I might go to Alcoholics Anonymous, but I think it would be difficult for me to remain anonymous'
message.attachments.add('george_best_quotes.txt')
message.save_draft()  # save the message on the cloud as a draft in the drafts folder
```

Working with saved emails is also easy:
```python
query = mailbox.new_query().on_attribute('subject').contains('george best')  # see Query object in Utils
messages = mailbox.get_messages(limit=25, query=query)

message = messages[0]  # get the first one

message.mark_as_read()
reply_msg = message.reply()

if 'example@example.com' in reply_msg.to:  # magic methods implemented
    reply_msg.body = 'George Best quote: I spent a lot of money on booze, birds and fast cars. The rest I just squandered.'
else:
    reply_msg.body = 'George Best quote: I used to go missing a lot... Miss Canada, Miss United Kingdom, Miss World.'

reply_msg.send()
```

## AddressBook
AddressBook groups the funcionality of both the Contact Folders and Contacts. Outlook Distribution Groups are not supported.

#### Contact Folders
Represents a Folder within your Contacts Section in Office 365.
AddressBook class represents the parent folder (it's a folder itself).

You can get any folder in your address book by requesting child folders or filtering by name.

```python
address_book = account.address_book()

contacts = address_book.get_contacts(limit=None)  # get all the contacts in the Personal Contacts root folder

work_contacts_folder = address_book.get_folder(folder_name='Work Contacts')  # get a folder with 'Work Contacts' name

message_to_all_contats_in_folder = work_contacts_folder.new_message()  # creates a draft message with all the contacts as recipients

message_to_all_contats_in_folder.subject = 'Hallo!'
message_to_all_contats_in_folder.body = """
George Best quote:

If you'd given me the choice of going out and beating four men and smashing a goal in
from thirty yards against Liverpool or going to bed with Miss World,
it would have been a difficult choice. Luckily, I had both.
"""
message_to_all_contats_in_folder.send()

# querying folders is easy:
child_folders = address_book.get_folders(25) # get at most 25 child folders

for folder in child_folders:
    print(folder.name, folder.parent_id)

# creating a contact folder:
address_book.create_child_folder('new folder')
```

#### The Global Address List
Office 365 API (Nor MS Graph API) has no concept such as the Outlook Global Address List.
However you can use the Users API to access all the users within your organization.

Without admin consent you can only access a few properties of each user such as name and email and litte more.
You can search by name or retrieve a contact specifying the complete email.

- Basic Permision needed is Users.ReadBasic.All (limit info)
- Full Permision is Users.Read.All but needs admin consent.

To search the Global Address List (Users API):

```python
global_address_list = account.address_book(address_book='gal')

# start a new query:
q = global_address_list.new_query('display_name')
q.startswith('George Best')

print(global_address_list.get_contacts(query=q))
```


To retrieve a contact by it's email:

```python
contact = global_address_list.get_contact_by_email('example@example.com')
```

#### Contacts
Everything returned from an `AddressBook` instance is a `Contact` instance.
Contacts have all the information stored as attributes

Creating a contact from an `AddressBook`:

```python
new_contact = address_book.new_contact()

new_contact.name = 'George Best'
new_contact.job_title = 'football player'
new_contact.emails.add('george@best.com')

new_contact.save()  # saved on the cloud

message = new_contact.new_message()  #  Bonus: send a message to this contact

# ...

new_contact.delete()  # Bonus: deteled from the cloud
```


## Calendar
The calendar and events functionality is group in a Schedule object.

A Schedule instance can list and create calendars. It can also list or create events on the default user calendar.
To use other calendars use a Calendar instance.  

Working with the Schedule instance:
```python
import datetime as dt

# ...
schedule = account.schedule()

new_event = schedule.new_event()  # creates a new event in the user default calendar
new_event.subject = 'Recruit George Best!'
new_event.location = 'England'

# naive datetimes will automatically be converted to timezone aware datetime
#  objects using the local timezone detected or the protocol provided timezone
new_event.start = dt.datetime(2018, 9, 5, 19, 45) 
# so new_event.start becomes: datetime.datetime(2018, 9, 5, 19, 45, tzinfo=<DstTzInfo 'Europe/Paris' CEST+2:00:00 DST>)

new_event.recurrence.set_daily(1, end=dt.datetime(2018, 9, 10))
new_event.remind_before_minutes = 45

new_event.save()
```

Working with Calendar instances:
```python
calendar = schedule.get_calendar(calendar_name='Birthdays')

calendar.name = 'Football players birthdays'
calendar.update()

q = calendar.new_query('start').ge(dt.datetime(2018, 5, 20)).chain('and').on_attribute('end').le(dt.datetime(2018, 5, 24))

birthdays = calendar.get_events(query=q)

for event in birthdays:
    if event.subject == 'George Best Birthday':
        # He died in 2005... but we celebrate anyway!
        event.accept("I'll attend!")  # send a response accepting
    else:
        event.decline("No way I'm comming, I'll be in England", send_response=False)  # decline the event but don't send a reponse to the organizer

```

## OneDrive
Work in progress


## Utils

#### Pagination

When using certain methods, it is possible that you request more items than the api can return in a single api call.
In this case the Api, returns a "next link" url where you can pull more data.

When this is the case, the methods in this library will return a `Pagination` object which abstracts all this into a single iterator.
The pagination object will request "next links" as soon as they are needed.

For example:

```python
maibox = account.mailbox()

messages = mailbox.get_messages(limit=1500)  # the Office 365 and MS Graph API have a 999 items limit returned per api call.

# Here messages is a Pagination instance. It's an Iterator so you can iterate over.

# The first 999 iterations will be normal list iterations, returning one item at a time.
# When the iterator reaches the 1000 item, the Pagination instance will call the api again requesting exactly 500 items
# or the items specified in the batch parameter (see later).

for message in messages:
    print(message.subject)
```

When using certain methods you will have the option to specify not only a limit option (the number of items to be returned) but a batch option.
This option will indicate the method to request data to the api in batches until the limit is reached or the data consumed.

For example:

```python
messages = mailbox.get_messages(limit=100, batch=25)

# messages here is a Pagination instance
# when iterating over it will call the api 4 times (each requesting 25 items).

for message in messages:  # 100 loops with 4 requests to the api server
    print(message.subject)
```

#### The Query helper

When using the Office 365 API you can filter some fields.
This filtering is tedious as is using [Open Data Protocol (OData)](http://docs.oasis-open.org/odata/odata/v4.0/errata03/os/complete/part2-url-conventions/odata-v4.0-errata03-os-part2-url-conventions-complete.html).

Every `ApiComponent` (such as `MailBox`) implements a new_query method that will return a `Query` instance.
This `Query` instance can handle the filtering (and sorting and selecting) very easily.

For example:

```python
query = mailbox.new_query()

query = query.on_attribute('subject').contains('george best').chain('or').startswith('quotes')

# 'created_date_time' will automatically be converted to the protocol casing.
# For example when using MS Graph this will become 'createdDateTime'.

query = query.chain('and').on_attribute('created_date_time').greater(datetime(2018, 3, 21))

print(query)

# contains(subject, 'george best') or startswith(subject, 'quotes') and createdDateTime gt '2018-03-21T00:00:00Z'
# note you can pass naive datetimes and those will be converted to you local timezone and then send to the api as UTC in iso8601 format

# To use Query objetcs just pass it to the query parameter:
filtered_messages = mailbox.get_messages(query=query)
```

#### Request Error Handling and Custom Errors

Whenever a Request error raises, the connection object will raise an exception.
Then the exception will be captured and logged it to the stdout with it's message, an return Falsy (None, False, [], etc...)

HttpErrors 4xx (Bad Request) and 5xx (Internal Server Error) are considered exceptions and raised also by the connection (you can configure this on the connection).

