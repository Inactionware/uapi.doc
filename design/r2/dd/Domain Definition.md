Application developer must define Domain for application.

# Define a Domain
A Domain is a plain Java class with some specific annotation:
```java
@Domain
public class User {
	
	@Field
	protected int id;
	
	@Field
	protected int name;
}
```

# Define Domain Relationship

```java
@Domain
public class Topic {

	@Field(mapTo="User.id", required=true)
	protected User author;
}

@Domain
public class User {
	
	@Field
	protected Topic[] topics;
}
```