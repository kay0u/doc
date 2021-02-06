---
title: Users groups and permissions
template: docs
taxonomy:
    category: docs
---

You can access the *groups and permissions* management interface from the webadmin
by going into the 'Users' section and clicking the corresponding button:

![](./images/button_to_go_to_permission_interface.png)

## Managing groups

The group mechanism can be used to define groups of users which then can be used to restrict permissions for applications and other services (such as mail or XMPP). Note that it is *not* mandatory to create a group to do so: you can also restrict access to an app or service on a user-per-user basis.

Using groups is however useful for semantics, for example if you host multiple groups of friends, associations or businesses on your server, you might want to create groups like `association1` and `association2` and add members of each association to the relevant group.

### Default groups

By default, two special groups are created:
- `all_users`, that contain all users registered on YunoHost,
- `visitors`, that applies to people viewing the server while not logged in. 

The content of those groups cannot be changed, only the permissions given to them.

### List existing groups

The existing groups are listed at the top of the *groups and permissions* page.

![](./images/groups_default-groups.png)

To list the currently existing groups in CLI :

```shell
$ yunohost user group list
groups:
  all_users:
    members:
      - alice
      - bob
      - charlie
      - delphine
```


### Creating a new group

To create a new group, simply click on the "New Group" button at the top of the page. You may only choose a name formed with letters (uper- and lowercase) and spaces. The group is created empty and without any permission.

![](./images/groups_button-new-group.png)

In CLI, to create a new group called `yolo_crew`

```shell
$ yunohost user group create yolo_crew
```

### Updating a group

Let's add a first to this group: in the group panel, click the button "add a user" and scroll to the desired user, then click on it.

![](./images/groups_button-add-user.png)

To remove a user, click on the cross next to their username, in the group panel.

![](./images/groups_button-remove-user.png)

In CLI, use the following command to add `charlie` and `delphine`to the `yolo_crew` group:

```shell
$ yunohost user group update yolo_crew --add charlie delphine
```

(similarly, `--remove` can be used to remove members from a group)

Now in the group list we should see:

```shell
$ yunohost user group list
groups:
  all_users:
    members:
      - alice
      - bob
      - charlie
      - delphine
  yolo_crew:
    members:
      - charlie
      - delphine
```

### Deleting groups

To delete a group, click on the red cross on the top right of the group panel. You will be asked for confirmation.

![](./images/groups_button-delete-group.png)

To delete the group `yolo_crew` in CLI, you may run

```shell
$ yunohost user group delete yolo_crew
```

## Managing permissions

The permission mechanism allow to restrict access to services (for example mail, XMPP...) and apps, or even specific parts of the apps (for example the administration interface of WordPress).

### List permissions

The groups page lists the permissions given to each group, including the special groups `all_users` and `visitors`.

![](./images/groups_default-with-permissions.png)

To list permissions and corresponding accesses in CLI:

```shell
$ yunohost user permission list
permissions:
  mail.main:
    allowed: all_users
  wordpress.admin:
    allowed:
  wordpress.main:
    allowed: all_users
  xmpp.main:
    allowed: all_users
```

Here, we find that all registered users can use email, XMPP, and access the WordPress blog. However, nobody can access the WordPress admin interface.

More details can be displayed by adding the `--full` option which will display the list of users corresponding to groups allowed, as well as urls associated to a permission (relevant for web apps).

### Add accesses to group or users

To add a permission to a group, simply click the "+" button in the group panel, scroll to the desired permission, then click on it.

![](./images/groups_add-permission-group.png)

To allow a group to access the WordPress admin interface in CLI:

```shell
$ yunohost user permission update wordpress.admin --add yolo_crew
```

Note that you can also allow a single user, by using the specific panel at the bottom of the page.

![](./images/groups_add-permission-user.png)

or in CLI:

```shell
$ yunohost user permission update wordpress.admin --add alice
```

And now we may see that both the YoloCrew and Alice have access to the WordPress admin interface:

```shell
$ yunohost user permission list
  [...]
  wordpress.admin:
    allowed:
      - yolo_crew
      - alice
  [...]
```

Note that, for example, if we want to restrict permission for email so that only Bob is allowed to email, we should also remove `all_users` from the permission, by deleting it from the `all_users` group panel, or in CLI:

```shell
$ yunohost user permission update mail --remove all_users --add bob
```

Note that for some reason the packager of the application can decide to protect a permission to avoid to add or remove the visitor of the permission. Generaly this because it make non sens for the admin to modify this permission.

The webadmin will issue a warning if you set a permission that is superseeded by a wider permission.

![](./images/groups_alerte-permission.png)

### Manage SSO tile

Since yunohost 3.8 you can add or remove a tile in the SSO. For each permission which has an url defined you can enable or disable the tile in the SSO. By example CLI you can enalble the tile for the permission `wordpress.admin`:

```shell
$ yunohost user permission update wordpress.admin --show_tile True
```


Notes for apps packagers
------------------------

Installing an app creates the permission `app.main` with `all_users` allowed by default.

If you wish to make the application publicly available, instead of the old `unprotected_urls` mechanism, you should give access to the special group `visitors`:

```shell
ynh_permission_update --permission "main" --add visitors
```

If you wish to create a custom permission for your app (e.g. to restrict access to an admin interface) you may use the following helpers:

```shell
ynh_permission_create --permission "admin" --url "/admin" --allowed "$admin_user" --label "Label for your permission"
```

You don't need to take care of removing permissions or backing up/restoring them as it is handled by the core of YunoHost.

### Migrating away from the legacy permission management

When migrating/fixing an app still using the legacy permission system, it should be understood that the accesses are now to be managed by features from the core, outside of the application scripts!

Application scripts are only expected to:
- if relevant, during the install script, initialize the main permission of the app as public (`visitors`) or private (`all_users`) or only accessible to specific groups/users ;
- if relevant, create and initialize any other specific permission (e.g. to some admin interface) in the install script (and *maybe* in some migration happening in the upgrade script).

Applications scripts should absolutely **NOT** mess up with any already-existing app accesses (including `unprotected`/`skipped_uris` settings) during any other case, as *it would reset any admin-defined access rule*!

When migrating away from the legacy permission, you should:
- remove any management of `$is_public`-like or `$admin_user`-like setting, except for any manifest question meant to either *initialize* the app as public/private or specific permissions ;
- remove any management of `skipped_`, `unprotected_` and `protected_uris` (and `_regex`) settings that are now considered obsolete and deprecated. (N.B.: you should **explicitly delete them in the upgrade script**). Instead, you should now rely on the new `ynh_permission_*` helpers instead. If you do feel like you still need to use them, please contact the core team to provide your feedback and we'll figure out something ;
For example, in the upgrade script if you used the `protected_uris` key before, you may use this code in the `DOWNWARD COMPATIBILITY` section:

```bash
protected_uris=$(ynh_app_setting_get --app=$app --key=protected_uris)

# Unused with the permission system
if [ ! -z "$protected_uris" ]; then
	ynh_app_setting_delete --app=$app --key=protected_uris
fi
```

- remove any call to `yunohost app addaccess` and similar actions that are now obsolete and deprecated.
- if your app use LDAP and support filter, use the filter `'(&(objectClass=posixAccount)(permission=cn=YOUR_APP.main,ou=permission,dc=yunohost,dc=org))'` to allow users who have this permission. (A complete documentation of LDAP [here](https://moulinette.readthedocs.io/en/latest/ldap.html) if you want to undestand how it works with YunoHost)

Here an example of how to migrate the code from legacy to new permission system: [example](https://github.com/YunoHost/example_ynh/pull/111/files)

#### Extended permission

Since yunohost 3.8 some new feature of permission has been integrated. Theses new feature should solve some issue with the permission system.

The new feature are:
- Tile and label support: You can now define a label for this permission. This label dedicated to the user. It will be shown on the webadmin to explain to the user what do this permission. If you set the parameter `show_tile` to `True` a new tile will be available on the SSO for the allowed user. This give the possiblity to have multiple tile on the SSO for the same app. The url of this tile will be the `url` this permission.
- Multiple url support: Now you can add multiple `url` for the same permission. This give the possiblity to protect many url with the same permission.
- Protection: If you don't want as packager that the admin play with the visitor group with a permission you can protect it. So the admin won't have the possiblity to add/remove the `visitor` group of this permission. Note that with the helper `ynh_permission_update` you keep the possibility to add/remove the `visitor` group of this permission.
- Auth header management: some app don't work with the auth header from SSOwat. You can know define for each permission if you want to set the auth header or not. To resume this folling array show the equivalent with permission of the old settings `protected_uris`, `unprotected_uris`, `skipped_uris`.

|             | with auth header | no auth header |
| :---------- | :--------------- | :------------- |
| **public**  | unprotected_uris | skipped_uris   |
| **private** | protected_uris   | not available  |

|             | with auth header                            | no auth header                               |
| :---------- | :------------------------------------------ | :------------------------------------------- |
| **public**  | auth_header=True, visitor group allowed     | auth_header=False, visitor group allowed     |
| **private** | auth_header=True, visitor group not allowed | auth_header=False, visitor group not allowed |


All of theses feature are managable by theses following helper:
- `ynh_permission_create`
- `ynh_permission_url`
- `ynh_permission_update`

If you have any question, please contact someone from the apps-group.
