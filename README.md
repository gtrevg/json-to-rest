json-to-rest
=================

# Overview

This document describes a pattern you may use in order to go from the data you're expecting to return and  manipulate to a REST url structure and methods to accomplish that task.  Data is the key.  Start with a good data definition and everything else will follow.


# User Model

Let's define three objects that are related in the user model space:  accounts, groups, and preferences.  These are the objects we are expecting to persist and manage via our REST interface


## Models

### Account

Here is an example account object that may get returned.


```json
{
  "href": "http://api.somewebsite.com/accounts/5678",
  "name": { "first": "Frankie", "last": "Manning" },
  "birthday": "1914-05-26",
  "photoUrl": "http://www.swingdancecouncil.com/hof_images/manning.jpg",
  "emails": [
  	"musclehead@frankiemanning.com",
  	"frankie@lindyhophistory.com"
  ],
  "phones": [
  	{ "type": "home", "id": "1", "number":"917-555-1212" }
  ],
  "groups": [
	{
      "href": "http://api.somewebsite.com/accounts/5678/groups/12YouKnowWhatToDo",
	  "id": "12YouKnowWhatToDo",
      "name": "Whitey's Lindy Hoppers",
      "joined": "1924-01-01T12:35:22Z",
      "role": "Administrator"
    }
  ]
}

```


### Group


```json
{
  "href": "http://api.somewebsite.com/groups/12YouKnowWhatToDo",
  "name": "Whitey's Lindy Hoppers",
  "created": "1924-01-01T12:35:22Z",
  "members": [
  	{
    	"href": "http://api.somewebsite.com/accounts/5678",
		"joined": "1924-01-01T12:35:22Z",
    	"role": "Administrator"
    }
  ]
}
```


### Preference

Preferences are tied to an account.

```json
{
  "href": "http://api.somewebsite.com/accounts/5678/preferences/swing.out",
  "name": "swing.out",
  "value": "Savoy Style"
}
```


## Goals

Given these models, the end goal is the same for every REST interface:  to allow people to create new objects, create new relations between objecdts, modify objecdts and their relations, delete objects and their relations, and find objects and their relations.  In the end, we want to provide a defined schema for each of these objects.

In addition, though, a REST API needs to additionally return meta information about the transaction.  For instance, returning response meta info should always be added in.  In addition, there are times when you are returning a collection of results and need to return info on the total number of items, the current offset, and current limit.  Because this additional meta info will always need to be included in responses, we will need to have a systematic way if including that information for both single items returned, as well as for collections.

Despite needing to include this meta information, we will still aim to keep the original objects as pure as possible so that  REST <-> JSON <-> Data Structures transformations are simple and repeatable.


## User Models to API Responses

The next step of the process is to be able to take your models and return them as either lists of objects (collections) or as the individual data items.  Collections will always need to support pagination.  The business requirement of how many items you actually end up storing is completely independent from the way clients gain access to those items.  If initially the business logic said to support 10 items, then later it became 10,000, not having a plan in place on how to paginate through that data would require significant API changes, as well as client changes.  Changes mean more bugs, more money, and more time.  It's better to design an interface that is forward compatible in this case, as the likelihood of that change coming down the pipeline is very high.  In addition, you can have one simple methodology for dealing with collections that need not change no matter how big your data set becomes.  Additionally, whether or not the API actually paginates the data can just be a configuration value (e.g.: limit=0)

### Basic API Response

For every response, some basic information will always be included:

```json
{
	"response": {
		"status": 200,
		"code": 1024,  // (if needed)
		"message": "success",
		"id": "ae9b538d-8a79-458b-a0b7-e2a08027b683"  // transaction id for tracing
	}
}
```

For requests that return a collection of results, there will be information related to pagingation.

```json
{
	"page": {
		"offset": 0,
		"limit": 100,
		"total": 34234
	}
}
```

Finally, the object or objects that have been requested will be in either the "item" or "items" key, depending on if it is a single item or collection respectively.


Single Item:

```json
{
	"item": {}
}
```

Collection:
```json
{
	"items": [],

	"page": {
		"offset": 0,
		"limit": 100,
		"total": 0
	}
}
```


### Single Item Response Example

To create an API response for a single item, add in response information, and add in the item under the "item" key.  For example:


```json
{
	"response": {
		"status": 200,
		"code": 1024,  // (if needed)
		"message": "success",
		"id": "ae9b538d-8a79-458b-a0b7-e2a08027b683"  // transaction id for tracing
	},

	"item": {
		"href": "http://api.somewebsite.com/accounts/5678",
		"name": { "first": "Frankie", "last": "Manning" },
		"birthday": "1914-05-26",
		"photoUrl": "http://www.swingdancecouncil.com/hof_images/manning.jpg",
		"emails": [
			"musclehead@frankiemanning.com",
			"frankie@lindyhophistory.com"
		],
		"phones": [
			{ "type": "home", "id": "1", "number":"917-555-1212" }
		],
		"groups": [
			{
			  "href": "http://api.somewebsite.com/groups/12YouKnowWhatToDo",
			  "name": "Whitey's Lindy Hoppers",
			  "joined": "1924-01-01T12:35:22Z",
			  "role": "Administrator"
			}
		]
	}
}
```

### Multiple Item Response Example

For a query on a collection, add in the list of results under the "items" key.  For example:

```json
{
	"response": {
		"status": 200,
		"code": 1024,  // (if needed)
		"message": "success",
		"id": "ae9b538d-8a79-458b-a0b7-e2a08027b683"  // transaction id for tracing
	},

	"page": {
		"offset": 0,
		"limit": 100,
		"total": 1
	},

	"items": [
		{
		  "href": "http://api.somewebsite.com/accounts/5678/preferences/swing.out",
		  "name": "swing.out",
		  "value": "Savoy Style"
		}
	]
}

```


## User Models to REST URLs and Methods

Objects in JSON have a rather simple structure.  There's only a few types of data values that it actually supports, and so this provides a simple framework for doing JSON -> REST tranformations.  Essentially there are three types to cover:  scalars, maps (objects), and sequences (collections).

Below is a table of the operations that make sense on each of these types of values:

| Type     | Transformations |
|----------|-----------------|
| scalar   | set             |
|          | get             |
|          | delete          |
| map      | setMap          |
|          | getMap          |
|          | deleteMap       |
|          | setKey          |
|          | getKey          |
|          | deleteKey       |
| sequence | setSequence     |
|          | getSequence     |
|          | deleteSequence  |
|          | appendItem      |
|          | deleteItem      |



### URL Mapping

The easiest way to come up with your URL is to actually use the JSON object keys in the object.  Then there is no decision as to what you should use.  It's all by convention.  For instance, if I wanted to access the phone numbers of the account object:

```json
{
  "href": "http://api.somewebsite.com/accounts/5678",
  "name": { "first": "Frankie", "last": "Manning" },
  "birthday": "1914-05-26",
  "photoUrl": "http://www.swingdancecouncil.com/hof_images/manning.jpg",
  "emails": [
  	"musclehead@frankiemanning.com",
  	"frankie@lindyhophistory.com"
  ],
  "phones": [
  	{ "type": "home", "id": "1", "number":"917-555-1212" }
  ],
  "groups": [
	{
      "href": "http://api.somewebsite.com/groups/12YouKnowWhatToDo",
      "name": "Whitey's Lindy Hoppers",
      "joined": "1924-01-01T12:35:22Z",
      "role": "Administrator"
    }
  ]
}
```

The URL would be the following: `http://api.website.com/v1/accounts/5678/phones`.

Accessing the first name of the account would be: `http://api.website.com/v1/accounts/5678/name/first`.

Accessing the type of the id=1 phone number would be: `http://api.website.com/v1/accounts/5678/phones/1/type`.

Accessing the group name would be: `http://api.website.com/v1/accounts/5678/groups/12YouKnowWhatToDo/name`.



### Scalars

Given a scalar value, there are essentially only three HTTP methods to use to manipulate the data:  PUT, GET, DELETE.  The URL used is based on the path to get to that actual object key (e.g. `accounts/5678/groups/12YouKnowWhatToDo/role` )


Going back to our original table of operations, we can now fill it in with the proper HTTP requests:

| Type     | Transformations | HTTP Request                                              |
|----------|-----------------|-----------------------------------------------------------|
| scalar   | set             | PUT http://api.com/v1/collection/{objectId}/scalarName    |
|          | get             | GET http://api.com/v1/collection/{objectId}/scalarName    |
|          | delete          | DELETE http://api.com/v1/collection/{objectId}/scalarName |


#### Setting Data

Because you've already drilled down to the actual attribute, the data that you should post no longer requires a key/value pairing.  Just submit the actual value as the post data and the API should set the value accordingly.

```text
curl -X PUT -d "1914-05-26" http://api.com/v1/accounts/5678/birthday
```

#### Getting Data

To get data, just issue a standard GET on the URL.  The result will be in the aforementioned "item" format, with one key/value pair.  For instance, the response to `curl http://api.com/v1/accounts/5678/birthday` would be:

```json
{
	"response": {
		"status": 200,
		"message": "success",
		"id": "ae9b538d-8a79-458b-a0b7-e2a08027b683"
	},

	"item": {
		"birthday": "1914-05-26"
	}
}
```

#### Deleting Data

To delete data, you can issue a simplet HTTP delete command in order to delete that particular attribute.  For exmaple:

```text
curl -X DELETE http://api.com/v1/accounts/5678/birthday
```


### Maps (Objects)

Maps (Objects) have six transformations that can be applied to them.

| Type     | Transformations |
|----------|-----------------|
| map      | setMap          |
|          | getMap          |
|          | deleteMap       |
|          | setKeys         |
|          | getKeys         |
|          | deleteKeys      |


PUT, GET, and DELETE can be used operate on a map as a whole.  In order to set, get, and modify keys, PATCH, GET, and DELETE should be used.

#### GetMap

The standard applies for getting a map.  It should be returned under the "item" object. `curl http://api.com/v1/accounts/5678` will end up returning the following structure:



```json
{
	"response": {
		"status": 200,
		"message": "success",
		"id": "ae9b538d-8a79-458b-a0b7-e2a08027b683"
	},

	"item": {
		"href": "http://api.somewebsite.com/accounts/5678",
		"name": { "first": "Frankie", "last": "Manning" },
		"birthday": "1914-05-26",
		"photoUrl": "http://www.swingdancecouncil.com/hof_images/manning.jpg",
		"emails": [
			"musclehead@frankiemanning.com",
			"frankie@lindyhophistory.com"
		],
		"phones": [
			{ "type": "home", "id": "1", "number":"917-555-1212" }
		],
		"groups": [
			{
			  "href": "http://api.somewebsite.com/groups/12YouKnowWhatToDo",
			  "name": "Whitey's Lindy Hoppers",
			  "joined": "1924-01-01T12:35:22Z",
			  "role": "Administrator"
			}
		]
	}
}
```

[TO BE COMPLETED]
