# v2 Auth and Security

## etcd Resources 
There are three types of resources in etcd

1. permission resources: users and roles in the user store
2. key-value resources: key-value pairs in the key-value store
3. settings resources: security settings, auth settings, and dynamic etcd cluster settings (election/heartbeat)

### Permission Resources 

#### Users
A user is an identity to be authenticated. Each user can have multiple roles. The user has a capability on the resource if one of the roles has that capability.

The special static `root` user has a ROOT role. (Caps for visual aid throughout)

#### Role
Each role has exact one associated Permission List. An permission list exists for each permission on key-value resources. A role with `manage` permission of a key-value resource can grant/revoke capability of that key-value to other roles.

The special static ROOT role has a full permissions on all key-value resources, the permission to manage user resources and settings resources. Only the ROOT role has the permission to manage user resources and modify settings resources.

#### Permissions

There are two types of permissions, `read` and `write`. All management stems from the ROOT user.

A Permission List is a list of allowed patterns for that particular permission (read or write). Only ALLOW prefixes (incidentally, this is what Amazon S3 does). DENY becomes more complicated and is TBD.

### Key-Value Resources
A key-value resource is a key-value pairs in the store. Given a list of matching patterns, permission for any given key in a request is granted if any of the patterns in the list match.

The glob match rules are as follows:

* `*` and `\` are special characters, representing "greedy match" and "escape" respectively.
  * As a corrolary, `\*` and `\\` are the corresponding literal matches. 
* All other bytes match exactly their bytes, starting always from the *first byte*. (For regex fans, `re.match` in Python) 
* Examples:
  * `/foo` matches only the single key/directory of `/foo`
  * `/foo*` matches the prefix `/foo`, and all subdirectories/keys
  * `/foo/*/bar` matches the keys bar in any (recursive) subdirectory of `/foo`.

### Settings Resources

Specific settings for the cluster as a whole. This can include adding and removing cluster members, enabling or disabling security, replacing certificates, and any other dynamic configuration by the administrator.

## v2 Auth

### Basic Auth
We only support [Basic Auth](http://en.wikipedia.org/wiki/Basic_access_authentication) for the first version. Client needs to attach the basic auth to the HTTP Authorization Header. 

### Authorization field for operations
Added to requests to /v2/keys, /v2/security
Add code 403 Forbidden to the set of responses from the v2 API
Authorization: Basic {encoded string}

### Future Work
Other types of auth can be considered for the future (eg, signed certs, public keys) but the `Authorization:` header allows for other such types

### Things out of Scope for etcd Permissions

* Pluggable AUTH backends like LDAP (other Authorization tokens generated by LDAP et al may be a possiblity)
* Very fine-grained access controls (eg: users modifying keys outside work hours)



## API endpoints

An Error JSON corresponds to:
{
  "name": "ErrErrorName",
  "description" : "The longer helpful description of the error."
}

#### Users

The User JSON object is formed as follows:

```
{
  "user": "userName"
  "password": "password"
  "roles": [
    "role1",
    "role2"
  ],
  "grant": [],
  "revoke": [],
}
```

Password is only passed when necessary. Last Modified is set by the server and ignored in all client posts.

**Get a list of users**

GET/HEAD  /v2/security/user

    Sent Headers:
        Authorization: Basic <BasicAuthString>
    Possible Status Codes:
        200 OK
        403 Forbidden
    200 Headers:
        Content-type: application/json
    200 Body:
        {
          "users": ["alice", "bob", "eve"]
        }

**Get User Details**

GET/HEAD  /v2/security/users/alice

    Sent Headers:
        Authorization: Basic <BasicAuthString>
    Possible Status Codes:
        200 OK
        403 Forbidden
        404 Not Found
    200 Headers:
        Content-type: application/json
    200 Body:
        {
          "user" : "alice"
          "roles" : ["fleet", "etcd"]
        }

**Create A User**

A user can be created with initial roles, if filled in. However, no roles are required; only the username and password fields

PUT  /v2/security/users/charlie

    Sent Headers:
        Authorization: Basic <BasicAuthString>
    Put Body:
        JSON struct, above, matching the appropriate name and with starting roles.
    Possible Status Codes:
        200 OK
        403 Forbidden
        409 Conflict (if exists)
    200 Body: (empty)

**Remove A User**

DELETE  /v2/security/users/charlie

    Sent Headers:
        Authorization: Basic <BasicAuthString>
    Possible Status Codes:
        200 OK
        403 Forbidden
        404 Not Found
    200 Headers:
    200 Body: (empty)

**Grant a Role(s) to a User**

PUT  /v2/security/users/charlie/grant

    Sent Headers:
        Authorization: Basic <BasicAuthString>
    Put Body:
        { "grantRoles" : ["fleet", "etcd"], (extra JSON data for checking OK) }
    Possible Status Codes:
        200 OK
        403 Forbidden
        404 Not Found
        409 Conflict
    200 Body: 
        JSON user struct, updated. "roles" now contains the grants, and "grantRoles" is empty. If there is an error in the set of roles to be added, for example, a non-existent role, then 409 is returned, with an error JSON stating why.

**Revoke a Role(s) from a User**

PUT  /v2/security/users/charlie/revoke

    Sent Headers:
        Authorization: Basic <BasicAuthString>
    Put Body:
        { "revokeRoles" : ["fleet"], (extra JSON data for checking OK) }
    Possible Status Codes:
        200 OK
        403 Forbidden
        404 Not Found
        409 Conflict
    200 Body: 
        JSON user struct, updated. "roles" now doesn't contain the roles, and "revokeRoles" is empty. If there is an error in the set of roles to be removed, for example, a non-existent role, then 409 is returned, with an error JSON stating why.

**Change password**

PUT  /v2/security/users/charlie/password

    Sent Headers:
        Authorization: Basic <BasicAuthString>
    Put Body:
        {"user": "charlie", "password": "newCharliePassword"}
    Possible Status Codes:
        200 OK
        403 Forbidden
        404 Not Found
    200 Body:
        JSON user struct, updated

#### Roles

A full role structure may look like this. A Permission List structure is used for the "permissions", "grant", and "revoke" keys.
```
{
  "role" : "fleet",
    "permissions" : {
      "kv" {
        "read" : [ "/fleet/" ],
        "write": [ "/fleet/" ],
      }
    }
    "grant" : {"kv": {...}},
    "revoke": {"kv": {...}},
    "members" : ["alice", "bob"],
}
```

**Get a list of Roles**

GET/HEAD  /v2/security/roles

    Sent Headers:
        Authorization: Basic <BasicAuthString>
    Possible Status Codes:
        200 OK
        403 Forbidden
    200 Headers:
        ETag: "<hash of list of roles>"
        Content-type: application/json
    200 Body:
        {
          "roles": ["fleet", "etcd", "quay"]
        }

**Get Role Details**

GET/HEAD  /v2/security/roles/fleet

    Sent Headers:
        Authorization: Basic <BasicAuthString>
    Possible Status Codes:
        200 OK
        403 Forbidden
        404 Not Found
    200 Headers:
        ETag: "roles/fleet:<lastModified>"
        Content-type: application/json
    200 Body:
        {
          "role" : "fleet",
          "read": {
            "prefixesAllowed": ["/fleet/"],
          },
          "write": {
            "prefixesAllowed": ["/fleet/"],
          },
        }

**Create A Role**

PUT  /v2/security/roles/rocket

    Sent Headers:
        Authorization: Basic <BasicAuthString>
    Put Body:
        Initial desired JSON state, complete with prefixes and 
    Possible Status Codes:
        201 Created
        403 Forbidden
        404 Not Found
        409 Conflict (if exists)
    200 Body: 
        JSON state of the role

**Remove A Role**

DELETE  /v2/security/roles/rocket

    Sent Headers:
        Authorization: Basic <BasicAuthString>
    Possible Status Codes:
        200 OK
        403 Forbidden
        404 Not Found
    200 Headers:
    200 Body: (empty)

**Update a Role’s Permission List for {read,write}ing**

PUT  /v2/security/roles/rocket/update

    Sent Headers:
        Authorization: Basic <BasicAuthString>
    Put Body:
        {
          "role" : "rocket",
          "grant": {
            "kv": {
              "read" : [ "/rocket/"]
            }
          }, 
          "revoke": {
            "kv": {
              "read" : [ "/fleet/"]
            }
          } 
        }
    Possible Status Codes:
        200 OK
        403 Forbidden
        404 Not Found
    200 Headers:
        ETag: "roles/rocket:<tzNow>"
    200 Body:
        JSON state of the role, with change containing empty lists and the deltas applied appropriately.


#### Enable and Disable Security
        
**Get security status**

GET  /v2/security/enable

    Sent Headers:
    Possible Status Codes:
        200 OK
    200 Body:
        {
          "enabled": true
        }


**Enable security**

Enabling security means setting an explicit `root` user and password. ROOTs roles are irrelevant, as this user has full permissions.

PUT  /v2/security/enable

    Sent Headers:
    Put Body:
        {
          "user" : "root",
          "password": "toor"
        }
    Possible Status Codes:
        200 OK
        400 Bad Request (if not a root user)
    200 Body: (empty)

**Disable security**

DELETE  /v2/security/enable

    Sent Headers:
        Authorization: Basic <RootAuthString>
    Possible Status Codes:
        200 OK
        403 Forbidden (if not a root user)
    200 Body: (empty)


## Example Workflow

Let's walk through an example to show two tenants (applications, in our case) using etcd permissions.

### Enable security

```
PUT  /v2/security/enable
    Headers:
    Put Body:
        {"user" : "root", "password": "root"}
```


### Change root's password

```
PUT  /v2/security/users/root/password
    Headers:
        Authorization: Basic <root:root>
    Put Body:
        {"user" : "root", "password": "betterRootPW!"}
```

//TODO(barakmich): How do you recover the root password? *This* may require a flag and a restart. `--disable-permissions`

### Create Roles for the Applications

Create the rocket role fully specified:

```
PUT /v2/security/roles/rocket
    Headers:
        Authorization: Basic <root:betterRootPW!>
    Body: 
        {
          "role" : "rocket",
          "permissions" : {
            "kv": {
              "read": [
                "/rocket/"
              ],
              "write": [
                "/rocket/"
              ]
            }
          }
        }
```

But let's make fleet just a basic role for now:

```
PUT /v2/security/roles/fleet
    Headers:
      Authorization: Basic <root:betterRootPW!>
    Body: 
        {
            "role" : "fleet",
        }
```

### Optional: Add some permissions to the roles

Well, we finally figured out where we want fleet to live. Let's fix it.
(Note that we avoided this in the rocket case. So this step is optional.)


```
PUT /v2/security/roles/fleet/update
    Headers:
        Authorization: Basic <root:betterRootPW!>
    Put Body:
        {
          "role" : "fleet",
          "grant" : {
            "kv" : {
              "read": [
                "/fleet/"
              ]
            }
          }
        }
```

### Create Users

Same as before, let's use rocket all at once and fleet separately

```
PUT /v2/security/users/rocketuser
    Headers:
        Authorization: Basic <root:betterRootPW!>
    Body:
        {"user" : "rocketuser", "password" : "rocketpw", "roles" : ["rocket"]}
```

```
PUT /v2/security/users/fleetuser
    Headers:
        Authorization: Basic <root:betterRootPW!>
    Body:
        {"user" : "fleetuser", "password" : "fleetpw"}
```

### Optional: Grant Roles to Users

Likewise, let's explicitly grant fleetuser access.

```
PUT /v2/security/users/fleetuser/grant
    Headers:
        Authorization: Basic <root:betterRootPW!>
    Body:
      {"user": "fleetuser", "grant": ["fleet"]}
```

#### Start to use fleetuser and rocketuser


For example:

```
PUT /v2/keys/rocket/RocketData
    Headers:
        Authorization: Basic <rocketuser:rocketpw>
```

Reads and writes outside the prefixes granted will fail with a 403 Forbidden.
