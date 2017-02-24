# Node URL Permission

This Node.js library facilitates formatting permissions for users, groups or any other security principal in the following way:

```
<namespace>:<identifier>:<privileges>
```

**For example:**

```js
import { permission } from 'resource-name-permissions';

if (!permission('us-east-1:article:read,create').allows('us-east-1:article:read')) {
  throw new Error('Not allowed here!');
}
```

# Description

Resource Name Permissions are a flexible method to express privileges in a succinct way.

They are inspired by [Github's OAauth Scopes](https://developer.github.com/changes/2014-02-24-finer-grained-scopes-for-ssh-keys/) and [Amazon's Resource Names](http://docs.aws.amazon.com/general/latest/gr/aws-arns-and-namespaces.html).

## Format

```
<namespace>:<identifier>:<privileges>
```

A Resource Name Permission consists of three components:

1. **Identifier**. The identifier specifies which resources the permission applies to. Valid characters are `a-Z`, `0-9`, `-`, `_`, `.` and `+`. Special characters are `/`, `:` and `*`.

    Both `/` and `:` serve the same purpose and enable you to hierarchically organize identifiers. Organizing identifiers is particularly useful in combination with wildcards. Whether you use `/` or `:` is entirely up to you, the `/` better matches URLs and a `:` mimics URNs. You can even combine the two.

    The `*` wildcard matches zero or any characters excluding `/` and `:`, whereas `**` matches zero or any characters including `/` and `:`.

    For example, to read a user comment with id `article/1234/comments/54` one of the following would grant access:

    ```
    article/1234/comments/54:read
    article/1234/comments/54:admin
    article/*/comment/*:read
    article/*/*/*:read
    article/**:read
    **:read
    ```

    These will NOT grant access:

    ```
    article:1234:comments:54:read
    article/1234/comments/54:update
    article/*:read
    ```

    Note that `**` is only valid when preceded and followed by a `/`, `:` or at the start of the permission. A normal wildcard `*` can be used anywhere within an identifier string.

2. **Privileges**. Privileges specify which operations are allowed on a resource. You can either specify these as a comma-separated set of names, a bitmask, or a combination thereof (e.g. `create,read,update,delete`, `15` and `crud` are equivalent). Privileges are fully customizable.

    The privileges configured by default are:

    Bitmask | Name            | Description
    --------|-----------------|-----------------
    `1`     | `read`          | View resource.
    `2`     | `create`        | Create resource.
    `4`     | `update`        | Update resource.
    `8`     | `delete`        | Delete resource.
    `15`    | `crud`          | All of the above.
    `16`    | `manage`        | Set permissions of subject without `manage` or `super` privilege.
    `31`    | `manager`       | All of the above.
    `32`    | `own`           | Set permissions of subjects without `super` privilege + allow ownership transfer.
    `63`    | `owner`         | All of the above.
    `64`    | `admin`         | Enables special administrative actions and setting permissions of everyone.
    `127`   | `administrator` | All of the above.

# permission(perm)

The `permission(perm)` function enables a variety of evaluation and transformation methods. To evaluate a collection of permissions scroll down until you hit the `permissions(perms[])` section.

## allows(searchPermissions[])

Returns `true` if all `searchPermissions` are matched by the permission, otherwise returns `false`.

```js
import { permission } from 'resource-name-permissions';

// Basic examples
permission('article:read').allows('article:read'); // true
permission('project-1:article:read').allows('project-1:article:read'); // true
permission('project-1:article:read').allows('article:read'); // false
permission('article:read,update').allows('article:read'); // true
permission('article:crud').allows('article:read,update'); // false
permission('article:read,update').allows('article:crud'); // false
permission('article:read').allows(['article:read', 'article:update']); // false
permission('article/1234:read').allows('article:read'); // false
permission('article:read').allows('article/1234:read'); // false
permission('article:read,update').allows('article:read', 'article:update'); // true
permission('article:read').allows('article:read', 'article:update'); // false

// Wildcards * and **
permission('art*:read').allows('article:read'); // true
permission('article/*:read').allows('article/1234:read'); // true
permission('article/*:read').allows('article:1234:read'); // false
permission('article/1234:read').allows('article/*:read'); // false
permission('article/*:read').allows('article:read'); // true
permission('article/*:read').allows('article/1234/comment:read'); // false
permission('article/**:read').allows('article/1234/comment:read'); // true
permission('article/**:read').allows('article/1234:comment:read'); // true
```

## identifier([ identifier ])

Gets or sets the permission identifier.

```js
const perm = permission('article/1234/comment/21:read');

perm.identifier(); // "article/1234/comment/21"
perm.path('article/998');
perm.path(); // "article/998"
```

## privileges([ privileges ])

Sets or returns priviliges

`privileges` can be either an array or comma-separated string with privilege names or their bitmasks. For example, `13`, `read`, `owner`, and `read,update,3` are valid.

When `privileges` are not defined method returns the bitmask of all privileges.

```js
const perm = permission('article/1234:read');

perm.privileges(); // 1
perm.privileges('crud,own');
perm.privileges(); // 15 | 32 = 47
perm.privileges(['crud', 'manage', 'owner']);
perm.privileges(); // 15 | 16 | 63 = 63
```

## hasPrivilege(privilege)

Return `true` when permission has specified privilege(s).

`privileges` can be either an array or comma-separated string with privilege names or their bitmasks.

```js
const perm = permission('article/1234:crud');

perm.hasPrivilege('read'); // true
perm.hasPrivilege(['read', 'create', 'update']); // true
perm.hasPrivilege('crud'); // true
perm.hasPrivilege('crud,read,create'); // true
perm.hasPrivilege('admin'); // false
perm.hasPrivilege('unknown'); // throws error
```

## hasPrivileges()

Alias of `hasPrivilege()`.

## grantPrivileges()

Gets array with privilege names that enable granting privileges.

Returns empty array when none of permission's privileges are grant privileges.

```js
const perm = permission('article/1234:read,manage,64');

perm.grantPrivileges(); // ['manage', 'admin']
```

## mayGrant(newPermission [, granteePermissions])

Returns `true` if permission allows granting `newPermission` to grantee.

In the grant process we distinguish two roles:

- *grantor*: the person granting or revoking a permission.
- *grantee*: the person receiving or losing a permission.

To grant a permission the grantor must have **manage** an/or **super** privilege.

- **manage** allows granting or revoking **crud** privileges from anyone without **manage** and **super** privilege.
- **super** allows granting and revoking permissions from anyone.

The grantor/grantee privileges can be customized using `permission.config()`.

```js
import { permission } from 'resource-name-permissions';

permission('article:manage').mayGrant('article:read', []); // true
permission('article:manage').mayGrant('article:read', ['article:delete']); // true
permission('article:manage').mayGrant('article:read', ['article:admin']); // true
permission('article:manage').mayGrant('article:manage', ['article:manage']); // false
permission('article:manage').mayGrant('article:read', ['unrelated:admin']); // true

permission('article:admin').mayGrant('article/1234:read', ['article:manage']); // true
permission('article:admin').mayGrant('article/1234:read', ['article:admin']); // true
```

## mayRevoke(permission [, granteePermissions])

Returns `true` if permission allows revoking `permission` to grantee.



```js
import { permission } from 'resource-name-permissions';

permission('article:manage').mayRevoke('article:read', []); // true
permission('article:manage').mayRevoke('article:read', ['article:admin']); // false
permission('article:manage').mayRevoke('article:manage', ['article:manage']); // false

permission('article:admin').mayRevoke('article/1234:read', ['article:manage']); // true
permission('article:admin').mayRevoke('article/1234:read', ['article:admin']); // true
```

## toObject()

Returns an object representation of an URL permission containing three properties: `url`, `attributes` and `privileges`.

```js
import { permission } from 'resource-name-permissions';

permission('article/*?author=user-1,user-2&flag=true:crud').toObject()
// {
//   path: 'article/*',
//   attributes: {
//     author: ['user-1', 'user-2'],
//     flag: ['true'],
//   },
//   privileges: 15,
// }
```

## toString()

Returns a string representation of an URL Permission. Privileges are always represented by their bitmask.

```js
import { permission } from 'resource-name-permissions';

permission('article/*?author=user-1:crud').toString()
// 'article/*?author=user-1:15'
```

## clone()

Returns a clone of the permission.

```js
const a = permission('article:read');
const b = a.clone();
a.privileges(['u']);
b.privileges(); // ['r']
```

Alternatively you can do this:

```js
const a = permission('article:read');
const b = permission(a); // Clone a
```

## validate()

Static method to checks if url permission string is valid.

```js
import { permission } from 'resource-name-permissions';

permission.validate('article:unknown'); // false, unknown privilege
```

## config(options)

Static method to change global config options. Changing global config doesn't affect existing instances.

```js
import { permission } from 'resource-name-permissions';

permission.config({
  privileges: {
    read: 1,
    create: 2,
    update: 4,
    delete: 8,
    crud: 15,
    manage: 16,
    manager: 31,
    own: 32,
    owner: 63,
    super: 64,
    admin: 127,
  },
  grantPrivileges: {
    manage: 15, // crud
    owner: 63, // owner + all the rest
    super: 127, // admin
  },
});
```

### Grant privileges

The config option `grantPrivileges` must be an object of key-value pairs where each key is a privilege name and each value a bitmask specifying which privileges it is allowed to grant.

Anyone with grant privileges is allowed to grant its privileges to grantees without any grant privileges. However, if a grantee does have grant privileges its grant privileges must be covered by those of the grantor.

Example:

```js
import { permission } from 'resource-name-permissions';

permission.config({
  privileges: {
    a: 1,
    x: 2,
    y: 4,
    z: 8,
  },
  grantPrivileges: {
    x: 1,
    y: 3, // 1 | 2 => a and x are allowed
    z: 9, // 1 | 8 => a and z are allowed
  },
});

permission('article:x').mayGrant('article:a'); // true
permission('article:x').mayGrant('article:a', ['article:x']); // false
permission('article:y').mayGrant('article:a', ['article:x']); // true
permission('article:y').mayGrant('article:x', ['article:x']); // true
permission('article:y').mayGrant('article:a', ['article:y']); // false
permission('article:z').mayGrant('article:a', ['article:z']); // true
```

Notice the last example where `z` may grant a permission to a grantee with `z`, whereas an `y` may not grant the same permission to another `y`. We'll leave it to you to figure out why.

# permissions(perms)

The `permissions(perms)` function enables verifying a collection of permissions.

Valid `perms` are permission strings, `permission()` objects, or arrays of either of those.

## allows(searchPermissions[])

Returns `true` if all `searchPermissions` are matched by any single or a combination of `permissions`.

This method intelligently handles privileges.

```js
import { permissions } from 'resource-name-permissions';

permissions('article:read', 'article:update').allows('article:ru'); // true
permissions('article/*:read', 'article/*:update').allows('article/1234:ru'); // true
```

## mayGrant(newPermission [, granteePermissions])

Returns `true` if any or a combination of this collection's permissions allow granting `newPermission` to grantee.

For a better understanding of how granting work, see `permission().mayGrant(...)`.

```js
import { permissions } from 'resource-name-permissions';

permission('article:read', 'article:m').mayGrant('article:read'); // true
```

## mayRevoke(removePermission [, granteePermissions])

An alias of `permissions().mayGrant()`.

Returns `true` if any or a combination of this collection's permissions allow revoking `removePermission` from grantee.

## permissions([ permissions ])

Gets or sets permissions.

`permissions` can be either a string or an array of permissions.