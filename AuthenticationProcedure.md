# Authentication Procedure

## Step 1, Admin User

### Creating an administrator user

```sql
INSERT INTO users (id, username, password, tin)
    VALUES (1, 'admin', '$2y$10$CkiuU47Gj57Mnh5pH.O5f.0GgxfNs1peNAWweCzVgv04xKT/zlrBW', '1');
```

- tin: ?
- passwordHash: "test" string encrypted using [bcrypt](https://bcrypt.online/)

### Create a role

```sql
INSERT INTO roles (id, name)
    VALUES (1, 'ROLE_ADMIN');
```

- names are found in `project.entity.ERole` (enum)
- probably should create all of them with a script before backend deployment

### Assign admin user to admin role

```sql
INSERT INTO user_roles (user_id, role_id)
    VALUES (1, 1);
```

- Now user `admin` has role `ROLE_ADMIN`

## Step 2, Obtaining auth token

### POST {{host}}/api/auth/signin

with a request body:

```json
{
 "username": "admin",
 "password": "test"
}
```

This request will return this response:

```json
{
    "id": 1,
    "username": "admin",
    "tin": "1",
    "roles": [
        "ROLE_ADMIN"
    ],
    "accessToken": "<token will be here>",
    "tokenType": "Bearer"
}
```

## Step 3, Use Admin Protected Routes

### Routes

From `project.config.SecurityConfig`

```java
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.cors().and().csrf().disable()
                .exceptionHandling().authenticationEntryPoint(unauthorizedHandler).and()
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS).and()
                .authorizeRequests().antMatchers("/api/auth/**").permitAll()
                .antMatchers("/api/admin/**").hasRole("ADMIN")  // <-- ADMIN ROUTE
                .antMatchers("/api/person/**").hasAnyRole("CITIZEN","NOTARY")
                .antMatchers("/api/declaration/**").hasAnyRole("CITIZEN","NOTARY")
                .antMatchers("/api/test/**").permitAll()
                .anyRequest().authenticated();

        http.addFilterBefore(authenticationJwtTokenFilter(), UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }
```

Now, an authorized admin can use:

| method | url | description |
| --- | --- | --- |
| GET | /api/admin/user/ | returns all registered users |
| POST | /api/admin/user/ | register a user |
| GET | /api/admin/user/{id} | get a user by id|
| DELETE | /api/admin/user/{id} | delete a user by id |
| POST | /api/admin/user/{id} | update a user's password by id |
| PUT | /api/admin/user/{id} | update a user by id |

### Creating a user

`URL`: POST {{host}}/api/admin/user

***IMPORTANT***, the request will need to contain an Authentication Header of bearer type.

This can be generally done in two ways through Postman:

1. Headers -> Key: Authentication -> Value: bearer \<token\>
2. Auth -> Type: Bearer Token -> Token: \<token\>

Also, you will need a request body:

```json
{
 "username": "test",
 "tin": "111111111",
 "password: "testtest"
}
```

This request will create a user with a `role: "ROLE_CITIZEN"` if the role attribute is not provided. However, for this automatic role to function it needs to exist in the `roles` table of the database.

To insert the role in the database run the following SQL

```sql
INSERT INTO roles (id, name)
    VALUES (2, 'ROLE_CITIZEN');
```

If any other roles are needed, add them to the database accordingly. Not sure if this has to happen through the admin's page of manually from the database. If this needs to be done manually, we should provide a script that creates all the roles needed for the application to function properly.

### Get all users

Remember to provide the token obtained throught the Sign In process, if you are an authorized admin then all of the registered users will be returned as a list of JSON objects
