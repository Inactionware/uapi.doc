# Schema

The UAPI based client using DIL to communicate with UAPI server, the DIL schema is simple, see below:
```
<Domain Operation>(<Operation Arguments>): <Operation Return>
```
It just like invoke specific operation on a Domain

## Domain Operation:

This part specify which Domain operation is invoked, it should be like:
```
<Domain Name>.<Operation Name>
```
The operation should be defined in the Domain.

## Operation Arguments:

The arguments of the operation should be contains two sections, one section is filter expression list, the second is assignment expression list, each filter or assignment expression is separated by `,`.

The filter use `:` as separator, like:
```
<Domain Property>:<value>
```

The assigement use `=` as separator, like:
```
<Domain Property>=<value>
```

## Operation Return:

The Operation Return specify which Domain will be returned by this operation, it likes:
```
<Domain Name> {
    <Domain Property List>
}
```


###############

The UIL (UAPI Interaction Language) supports query, update, delete on specific Domain or on multiple Domain object.

In this article, we will use below Domain in all example:
```
User {
	id: int,
	name: string,
}

Topic {
	id: int,
	title: string,
	content: string,
	author: User,
	replies: Reply[]
}

Reply {
	id: int,
	content: string,
	author: User,
	topic: Topic
}
```

# Domain Query

## Single Domain based Query
Schema:
```uil
<Domain>.<Domain Operation>(<Filters>): <Domain> {
	<Domain Fields>
}
```
* `Domain` is a Domain name, `Domain Operation` is an operation which is defined in the Domain.
* `Filters` is condition list to filter query result, it should be a `Domain Field:value` pattern.
* The `Domain` after `:` is the Domain type that Domain operation returned, in general it can be ignored
* The content between `{}` is used to specify the fields list, the fields must be defined in the `Domain`.
`Domain Fields` is Domain field list which list the query result will return 

Example:
We query out all topic which is published by user 'Mike':
```uil
Topic.list(author='Mike'): Topic {}
```
Since no fields is specified between `{}`, so the query will return all fields but relevant fields, see next section for more details on relevant Domain fields.

## Relevant Domain based Query
The schema is same as single Domain based query, the differant thing is the  `Domain Fields` supports specify relevant Domain fields.

For example, we want to query all topic and topic's replies, like below:
```uil
Topic.list(): Topic {
	topicId,
	title,
	content,
	author: User {
		name
	},
	replies: Reply {
		replyId,
		content
	}
}
```

# Domain Updating
## Update Single Domain Object

```uil
Topic.update(id:34, title='New Title'): Topic {
	...
}
```

## Update Multiple Domain Object
```uil
Topic.update({id:34, title='New Title'}, {id:35, title='New Title'}): {
	...
}
```

## Update batch Domain Object
```
Topic.update({author.name:'Mike', author.name='Joe'}): Topic {
	...
}
```

# Domain Deleting
Delete topic by spefic id
```uil
Topic.delete(id:[12,23]): Topic {
	...
}
```

# Relevant Domain Removing

```uil
Topic.remove(author.name:'Mike')
```