Application developer must define Domain for application.

# Define a Domain
A Domain is a plain Java class with some specific annotations:
```java
@Domain
public class User {
	
	@Property
	protected int id;
	
	@Property
	protected int name;
}
```

# Define Domain Relationship

## One-One Relationship

For example a User Domain has a extend Domain to define additional user information:
```java
@Domain
public class UserDetail {

	... other fields
	
	@Property
	protected User user;
}

@Domain
public class User {

	... other fields
	
	@Property
	protected UserDetail userDetail;
}
```

## One-Many Relationship

```java
@Domain
public class Role {

	@Property
	protected User author;
}

@Domain
public class User {
	
	@Property
	protected Topic[] topics;
}
```

## Many-Many Relation

```java
@Domain
public class Permisssion {

	@Property
	protected int id;

	@Property
	protected Role[] roles;
}

@Domain
public class Role {

	@Property
	protected int id;

	@Property
	protected Permission[] permissions;
}
```
