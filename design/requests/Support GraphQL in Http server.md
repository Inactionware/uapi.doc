## Proposal
GraphQL is more powerful and flexible than REST.  
The target of this feature are:

*   Integrate with protocol framework.
*   Integrate with resource based authentication framework.
*   GraphQL schema should be based on POJO's annotation.
*   The annotation based GraphQL information can be export to GraphQL SDL file.

## Solution

*   Use Java annotation to define GraphQL schema:

For instance the resource class is:

```java
@Resource
public class User {

    private String _id;

    private String _name;

    private Role[] roles;
}

@Resource
public class Role {

    private String _id;

    private String _name;
}
```

The GQL mapping is:

```java
public class GqlField {

    String _name;

    Class<?> _type;
}

public interface IGqlType {

    default String fieldPrefix() {
        return "";
    }

   Class<?> resourceType();

    GqlField[] fields();
}

@GqlMapping(User.class)
public abstract class GqlUser implements IGqlType{ }
```

The annotation processor will generate Gql Type class like below:

```java
public final class GqlRole_Generated extends GqlRole {

    private final GqlField[] _fields;

    public GqlUser_Generated() {
        this._fields = new GqlField[] {
            new GqlField("id", String.class),
            new GqlField("name", String.class)
        };
    }

    @Override
    public Class<User> resourceType() {
        return User.class;
    }

    @Override
    public GqlField[] fields() {
        return this._fields;
    }
}

public final class GqlUser_Generated extends GqlUser {

    private final GqlField[] _fields;

    public GqlUser_Generated() {
        this._fields = new GqlField[] {
            new GqlField("id", String.class),
            new GqlField("name", String.class),
            new GqlField("roles", GqlRole_Generated.class)
        };
    }

    @Override
    public Class<User> resourceType() {
        return User.class;
    }

    @Override
    public GqlField[] fields() {
        return this._fields;
    }
}
```

## Q&A
TODO

## Note
TODO