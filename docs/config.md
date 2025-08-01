# Configuring the ntfy server
The ntfy server can be configured in three ways: using a config file (typically at `/etc/ntfy/server.yml`, 
see [server.yml](https://github.com/binwiederhier/ntfy/blob/main/server/server.yml)), via command line arguments 
or using environment variables.

## Quick start
By default, simply running `ntfy serve` will start the server at port 80. No configuration needed. Batteries included 😀. 
If everything works as it should, you'll see something like this:
```
$ ntfy serve
2021/11/30 19:59:08 Listening on :80
```

You can immediately start [publishing messages](publish.md), or subscribe via the [Android app](subscribe/phone.md),
[the web UI](subscribe/web.md), or simply via [curl or your favorite HTTP client](subscribe/api.md). To configure 
the server further, check out the [config options table](#config-options) or simply type `ntfy serve --help` to
get a list of [command line options](#command-line-options).

## Example config
!!! info
    Definitely check out the **[server.yml](https://github.com/binwiederhier/ntfy/blob/main/server/server.yml)** file. It contains examples and detailed descriptions of all the settings.
    You may also want to look at how ntfy.sh is configured in the [ntfy-ansible](https://github.com/binwiederhier/ntfy-ansible) repository.

The most basic settings are `base-url` (the external URL of the ntfy server), the HTTP/HTTPS listen address (`listen-http`
and `listen-https`), and socket path (`listen-unix`). All the other things are additional features.

Here are a few working sample configs using a `/etc/ntfy/server.yml` file:

=== "server.yml (HTTP-only, with cache + attachments)"
    ``` yaml
    base-url: "http://ntfy.example.com"
    cache-file: "/var/cache/ntfy/cache.db"
    attachment-cache-dir: "/var/cache/ntfy/attachments"
    ```

=== "server.yml (HTTP+HTTPS, with cache + attachments)"
    ``` yaml
    base-url: "http://ntfy.example.com"
    listen-http: ":80"
    listen-https: ":443"
    key-file: "/etc/letsencrypt/live/ntfy.example.com.key"
    cert-file: "/etc/letsencrypt/live/ntfy.example.com.crt"
    cache-file: "/var/cache/ntfy/cache.db"
    attachment-cache-dir: "/var/cache/ntfy/attachments"
    ```

=== "server.yml (behind proxy, with cache + attachments)"
    ``` yaml
    base-url: "http://ntfy.example.com"
    listen-http: ":2586"
    cache-file: "/var/cache/ntfy/cache.db"
    attachment-cache-dir: "/var/cache/ntfy/attachments"
    behind-proxy: true
    ```

=== "server.yml (ntfy.sh config)"
    ``` yaml
    # All the things: Behind a proxy, Firebase, cache, attachments, 
    # SMTP publishing & receiving

    base-url: "https://ntfy.sh"
    listen-http: "127.0.0.1:2586"
    firebase-key-file: "/etc/ntfy/firebase.json"
    cache-file: "/var/cache/ntfy/cache.db"
    behind-proxy: true
    attachment-cache-dir: "/var/cache/ntfy/attachments"
    smtp-sender-addr: "email-smtp.us-east-2.amazonaws.com:587"
    smtp-sender-user: "AKIDEADBEEFAFFE12345"
    smtp-sender-pass: "Abd13Kf+sfAk2DzifjafldkThisIsNotARealKeyOMG."
    smtp-sender-from: "ntfy@ntfy.sh"
    smtp-server-listen: ":25"
    smtp-server-domain: "ntfy.sh"
    smtp-server-addr-prefix: "ntfy-"
    keepalive-interval: "45s"
    ```

Alternatively, you can also use command line arguments or environment variables to configure the server. Here's an example
using Docker Compose (i.e. `docker-compose.yml`):

=== "Docker Compose (w/ auth, cache, attachments)"
    ``` yaml
	services:
	  ntfy:
	    image: binwiederhier/ntfy
	    restart: unless-stopped
	    environment:
	      NTFY_BASE_URL: http://ntfy.example.com
	      NTFY_CACHE_FILE: /var/lib/ntfy/cache.db
	      NTFY_AUTH_FILE: /var/lib/ntfy/auth.db
	      NTFY_AUTH_DEFAULT_ACCESS: deny-all
	      NTFY_AUTH_USERS: 'phil:$2a$10$YLiO8U21sX1uhZamTLJXHuxgVC0Z/GKISibrKCLohPgtG7yIxSk4C:admin'
	      NTFY_BEHIND_PROXY: true
	      NTFY_ATTACHMENT_CACHE_DIR: /var/lib/ntfy/attachments
	      NTFY_ENABLE_LOGIN: true
	    volumes:
	      - ./:/var/lib/ntfy
	    ports:
	      - 80:80
	    command: serve
    ```

=== "Docker Compose (w/ auth, cache, web push, iOS)"
    ``` yaml
	services:
	  ntfy:
	    image: binwiederhier/ntfy
	    restart: unless-stopped
	    environment:
	      NTFY_BASE_URL: http://ntfy.example.com
	      NTFY_CACHE_FILE: /var/lib/ntfy/cache.db
	      NTFY_AUTH_FILE: /var/lib/ntfy/auth.db
	      NTFY_AUTH_DEFAULT_ACCESS: deny-all
	      NTFY_BEHIND_PROXY: true
	      NTFY_ATTACHMENT_CACHE_DIR: /var/lib/ntfy/attachments
	      NTFY_ENABLE_LOGIN: true
	      NTFY_UPSTREAM_BASE_URL: https://ntfy.sh
	      NTFY_WEB_PUSH_PUBLIC_KEY: <public_key>
	      NTFY_WEB_PUSH_PRIVATE_KEY: <private_key>
	      NTFY_WEB_PUSH_FILE: /var/lib/ntfy/webpush.db
	      NTFY_WEB_PUSH_EMAIL_ADDRESS: <email>
	    volumes:
	      - ./:/var/lib/ntfy
	    ports:
	      - 8093:80
	    command: serve
    ```

## Message cache
If desired, ntfy can temporarily keep notifications in an in-memory or an on-disk cache. Caching messages for a short period
of time is important to allow [phones](subscribe/phone.md) and other devices with brittle Internet connections to be able to retrieve
notifications that they may have missed. 

By default, ntfy keeps messages **in-memory for 12 hours**, which means that **cached messages do not survive an application
restart**. You can override this behavior using the following config settings:

* `cache-file`: if set, ntfy will store messages in a SQLite based cache (default is empty, which means in-memory cache).
  **This is required if you'd like messages to be retained across restarts**.
* `cache-duration`: defines the duration for which messages are stored in the cache (default is `12h`). 

You can also entirely disable the cache by setting `cache-duration` to `0`. When the cache is disabled, messages are only
passed on to the connected subscribers, but never stored on disk or even kept in memory longer than is needed to forward
the message to the subscribers.

Subscribers can retrieve cached messaging using the [`poll=1` parameter](subscribe/api.md#poll-for-messages), as well as the
[`since=` parameter](subscribe/api.md#fetch-cached-messages).

## Attachments
If desired, you may allow users to upload and [attach files to notifications](publish.md#attachments). To enable
this feature, you have to simply configure an attachment cache directory and a base URL (`attachment-cache-dir`, `base-url`). 
Once these options are set and the directory is writable by the server user, you can upload attachments via PUT.

By default, attachments are stored in the disk-cache **for only 3 hours**. The main reason for this is to avoid legal issues
and such when hosting user controlled content. Typically, this is more than enough time for the user (or the auto download 
feature) to download the file. The following config options are relevant to attachments:

* `base-url` is the root URL for the ntfy server; this is needed for the generated attachment URLs
* `attachment-cache-dir` is the cache directory for attached files
* `attachment-total-size-limit` is the size limit of the on-disk attachment cache (default: 5G)
* `attachment-file-size-limit` is the per-file attachment size limit (e.g. 300k, 2M, 100M, default: 15M)
* `attachment-expiry-duration` is the duration after which uploaded attachments will be deleted (e.g. 3h, 20h, default: 3h)

Here's an example config using mostly the defaults (except for the cache directory, which is empty by default): 

=== "/etc/ntfy/server.yml (minimal)"
    ``` yaml
    base-url: "https://ntfy.sh"
    attachment-cache-dir: "/var/cache/ntfy/attachments"
    ```

=== "/etc/ntfy/server.yml (all options)"
    ``` yaml
    base-url: "https://ntfy.sh"
    attachment-cache-dir: "/var/cache/ntfy/attachments"
    attachment-total-size-limit: "5G"
    attachment-file-size-limit: "15M"
    attachment-expiry-duration: "3h"
    visitor-attachment-total-size-limit: "100M"
    visitor-attachment-daily-bandwidth-limit: "500M"
    ```

Please also refer to the [rate limiting](#rate-limiting) settings below, specifically `visitor-attachment-total-size-limit`
and `visitor-attachment-daily-bandwidth-limit`. Setting these conservatively is necessary to avoid abuse.

## Access control
By default, the ntfy server is open for everyone, meaning **everyone can read and write to any topic** (this is how
ntfy.sh is configured). To restrict access to your own server, you can optionally configure authentication and authorization. 

ntfy's auth is implemented with a simple [SQLite](https://www.sqlite.org/)-based backend. It implements two roles 
(`user` and `admin`) and per-topic `read` and `write` permissions using an [access control list (ACL)](https://en.wikipedia.org/wiki/Access-control_list). 
Access control entries can be applied to users as well as the special everyone user (`*`), which represents anonymous API access. 

To set up auth, **configure the following options**:

* `auth-file` is the user/access database; it is created automatically if it doesn't already exist; suggested 
  location `/var/lib/ntfy/user.db` (easiest if deb/rpm package is used)
* `auth-default-access` defines the default/fallback access if no access control entry is found; it can be
  set to `read-write` (default), `read-only`, `write-only` or `deny-all`. **If you are setting up a private instance,
  you'll want to set this to `deny-all`** (see [private instance example](#example-private-instance)).

Once configured, you can use 

- the `ntfy user` command and the `auth-users` config option to [add or modify users](#users-and-roles)
- the `ntfy access` command and the `auth-access` option to [modify the access control list](#access-control-list-acl)
and topic patterns, and
- the `ntfy token` command and the `auth-tokens` config option to [manage access tokens](#access-tokens) for users.

These commands **directly edit the auth database** (as defined in `auth-file`), so they only work on the server, 
and only if the user accessing them has the right permissions.

### Users and roles
Users can be added to the ntfy user database in two different ways

* [Using the CLI](#users-via-the-cli): Using the `ntfy user` command, you can manually add/update/remove users.
* [In the config](#users-via-the-config): You can provision users in the `server.yml` file via `auth-users` key.

#### Users via the CLI
The `ntfy user` command allows you to add/remove/change users in the ntfy user database, as well as change
passwords or roles (`user` or `admin`). In practice, you'll often just create one admin 
user with `ntfy user add --role=admin ...` and be done with all this (see [example below](#example-private-instance)).

**Roles:**

* Role `user` (default): Users with this role have no special permissions. Manage access using `ntfy access`
  (see [below](#access-control-list-acl)).
* Role `admin`: Users with this role can read/write to all topics. Granular access control is not necessary.

**Example commands** (type `ntfy user --help` or `ntfy user COMMAND --help` for more details):

```
ntfy user list                     # Shows list of users (alias: 'ntfy access')
ntfy user add phil                 # Add regular user phil  
ntfy user add --role=admin phil    # Add admin user phil
ntfy user del phil                 # Delete user phil
ntfy user change-pass phil         # Change password for user phil
ntfy user change-role phil admin   # Make user phil an admin
ntfy user change-tier phil pro     # Change phil's tier to "pro"
ntfy user hash                     # Generate password hash, use with auth-users config option
```

#### Users via the config
As an alternative to manually creating users via the `ntfy user` CLI command, you can provision users declaratively in
the `server.yml` file by adding them to the `auth-users` array. This is useful for general admins, or if you'd like to
deploy your ntfy server via Docker/Ansible without manually editing the database.

The `auth-users` option is a list of users that are automatically created/updated when the server starts. Users
previously defined in the config but later removed will be deleted. Each entry is defined in the format `<username>:<password-hash>:<role>`.

Here's an example with two users: `phil` is an admin, `ben` is a regular user.

=== "Declarative users in /etc/ntfy/server.yml"
    ``` yaml
    auth-file: "/var/lib/ntfy/user.db"
    auth-users:
      - "phil:$2a$10$YLiO8U21sX1uhZamTLJXHuxgVC0Z/GKISibrKCLohPgtG7yIxSk4C:admin"
      - "ben:$2a$10$NKbrNb7HPMjtQXWJ0f1pouw03LDLT/WzlO9VAv44x84bRCkh19h6m:user"
    ```

=== "Declarative users via env variables"
    ```
    # Comma-separated list, use single quotes to avoid issues with the bcrypt hash 
    NTFY_AUTH_FILE='/var/lib/ntfy/user.db'
    NTFY_AUTH_USERS='phil:$2a$10$YLiO8U21sX1uhZamTLJXHuxgVC0Z/GKISibrKCLohPgtG7yIxSk4C:admin,ben:$2a$10$NKbrNb7HPMjtQXWJ0f1pouw03LDLT/WzlO9VAv44x84bRCkh19h6m:user'
    ```

The password hash can be created using `ntfy user hash` or an [online bcrypt generator](https://bcrypt-generator.com/) (though
note that you're putting your password in an untrusted website).

!!! important
    Users added declaratively via the config file are marked in the database as "provisioned users". Removing users
    from the config file will **delete them from the database** the next time ntfy is restarted.

    Also, users that were originally manually created will be "upgraded" to be provisioned users if they are added to
    the config. Adding a user manually, then adding it to the config, and then removing it from the config will hence
    lead to the **deletion of that user**.

### Access control list (ACL)
The access control list (ACL) **manages access to topics for non-admin users, and for anonymous access (`everyone`/`*`)**.
Each entry represents the access permissions for a user to a specific topic or topic pattern. Entries can be created in
two different ways:

* [Using the CLI](#acl-entries-via-the-cli): Using the `ntfy access` command, you can manually edit the access control list.
* [In the config](#acl-entries-via-the-config): You can provision ACL entries in the `server.yml` file via `auth-access` key.

#### ACL entries via the CLI
The ACL can be displayed or modified with the `ntfy access` command:

```
ntfy access                            # Shows access control list (alias: 'ntfy user list')
ntfy access USERNAME                   # Shows access control entries for USERNAME
ntfy access USERNAME TOPIC PERMISSION  # Allow/deny access for USERNAME to TOPIC
```

A `USERNAME` is an existing user, as created with `ntfy user add` (see [users and roles](#users-and-roles)), or the 
anonymous user `everyone` or `*`, which represents clients that access the API without username/password.

A `TOPIC` is either a specific topic name (e.g. `mytopic`, or `phil_alerts`), or a wildcard pattern that matches any
number of topics (e.g. `alerts_*` or `ben-*`). Only the wildcard character `*` is supported. It stands for zero to any 
number of characters.

A `PERMISSION` is any of the following supported permissions:

* `read-write` (alias: `rw`): Allows [publishing messages](publish.md) to the given topic, as well as 
  [subscribing](subscribe/api.md) and reading messages
* `read-only` (aliases: `read`, `ro`): Allows only subscribing and reading messages, but not publishing to the topic
* `write-only` (aliases: `write`, `wo`): Allows only publishing to the topic, but not subscribing to it
* `deny` (alias: `none`): Allows neither publishing nor subscribing to a topic 

**Example commands** (type `ntfy access --help` for more details):
```
ntfy access                        # Shows entire access control list
ntfy access phil                   # Shows access for user phil
ntfy access phil mytopic rw        # Allow read-write access to mytopic for user phil
ntfy access everyone mytopic rw    # Allow anonymous read-write access to mytopic
ntfy access everyone "up*" write   # Allow anonymous write-only access to topics "up..."
ntfy access --reset                # Reset entire access control list
ntfy access --reset phil           # Reset all access for user phil
ntfy access --reset phil mytopic   # Reset access for user phil and topic mytopic
```

**Example ACL:**
```
$ ntfy access
user phil (admin)
- read-write access to all topics (admin role)
user ben (user)
- read-write access to topic garagedoor
- read-write access to topic alerts*
- read-only access to topic furnace
user * (anonymous)
- read-only access to topic announcements
- read-only access to topic server-stats
- no access to any (other) topics (server config)
```

In this example, `phil` has the role `admin`, so he has read-write access to all topics (no ACL entries are necessary).
User `ben` has three topic-specific entries. He can read, but not write to topic `furnace`, and has read-write access
to topic `garagedoor` and all topics starting with the word `alerts` (wildcards). Clients that are not authenticated
(called `*`/`everyone`) only have read access to the `announcements` and `server-stats` topics.

#### ACL entries via the config
As an alternative to manually creating ACL entries via the `ntfy access` CLI command, you can provision access control
entries declaratively in the `server.yml` file by adding them to the `auth-access` array, similar to the `auth-users` 
option (see [users via the config](#users-via-the-config).

The `auth-access` option is a list of access control entries that are automatically created/updated when the server starts.
When entries are removed, they are deleted from the database. Each entry is defined in the format `<username>:<topic-pattern>:<access>`.

The `<username>` can be any existing, provisioned user as defined in the `auth-users` section (see [users via the config](#users-via-the-config)),
or `everyone`/`*` for anonymous access. The `<topic-pattern>` can be a specific topic name or a pattern with wildcards (`*`). The 
`<access>` can be one of the following:

* `read-write` or `rw`: Allows both publishing to and subscribing to the topic
* `read-only`, `read`, or `ro`: Allows only subscribing to the topic
* `write-only`, `write`, or `wo`: Allows only publishing to the topic
* `deny-all`, `deny`, or `none`: Denies all access to the topic

Here's an example with several ACL entries:

=== "Declarative ACL entries in /etc/ntfy/server.yml"
    ``` yaml
    auth-file: "/var/lib/ntfy/user.db"
    auth-users:
      - "phil:$2a$10$YLiO8U21sX1uhZamTLJXHuxgVC0Z/GKISibrKCLohPgtG7yIxSk4C:user"
      - "ben:$2a$10$NKbrNb7HPMjtQXWJ0f1pouw03LDLT/WzlO9VAv44x84bRCkh19h6m:user"
    auth-access:
      - "phil:mytopic:rw"
      - "ben:alerts-*:rw"
      - "ben:system-logs:ro"
      - "*:announcements:ro" # or: "everyone:announcements,ro"
    ```

=== "Declarative ACL entries via env variables"
    ```
    # Comma-separated list
    NTFY_AUTH_FILE='/var/lib/ntfy/user.db'
    NTFY_AUTH_USERS='phil:$2a$10$YLiO8U21sX1uhZamTLJXHuxgVC0Z/GKISibrKCLohPgtG7yIxSk4C:user,ben:$2a$10$NKbrNb7HPMjtQXWJ0f1pouw03LDLT/WzlO9VAv44x84bRCkh19h6m:user'
    NTFY_AUTH_ACCESS='phil:mytopic:rw,ben:alerts-*:rw,ben:system-logs:ro,*:announcements:ro'
    ```

In this example, the `auth-users` section defines two users, `phil` and `ben`. The `auth-access` section defines
access control entries for these users. `phil` has read-write access to the topic `mytopic`, while `ben` has read-write
access to all topics starting with `alerts-` and read-only access to the topic `system-logs`. The last entry allows
anonymous users (i.e. clients that do not authenticate) to read the `announcements` topic.

### Access tokens
In addition to username/password auth, ntfy also provides authentication via access tokens. Access tokens are useful
to avoid having to configure your password across multiple publishing/subscribing applications. For instance, you may
want to use a dedicated token to publish from your backup host, and one from your home automation system.

!!! info
    As of today, access tokens grant users **full access to the user account**. Aside from changing the password,
    and deleting the account, every action can be performed with a token. Granular access tokens are on the roadmap,
    but not yet implemented.

You can create access tokens in two different ways:

* [Using the CLI](#tokens-via-the-cli): Using the `ntfy token` command, you can manually add/update/remove tokens.
* [In the config](#tokens-via-the-config): You can provision access tokens in the `server.yml` file via `auth-tokens` key.

#### Tokens via the CLI
The `ntfy token` command can be used to manage access tokens for users. Tokens can have labels, and they can expire
automatically (or never expire). Each user can have up to 60 tokens (hardcoded). 

**Example commands** (type `ntfy token --help` or `ntfy token COMMAND --help` for more details):
```
ntfy token list                      # Shows list of tokens for all users
ntfy token list phil                 # Shows list of tokens for user phil
ntfy token add phil                  # Create token for user phil which never expires
ntfy token add --expires=2d phil     # Create token for user phil which expires in 2 days
ntfy token remove phil tk_th2sxr...  # Delete token
ntfy token generate                  # Generate random token, can be used in auth-tokens config option
```

**Creating an access token:**
```
$ ntfy token add --expires=30d --label="backups" phil
$ ntfy token list
user phil
- tk_7eevizlsiwf9yi4uxsrs83r4352o0 (backups), expires 15 Mar 23 14:33 EDT, accessed from 0.0.0.0 at 13 Feb 23 13:33 EST
```

Once an access token is created, you can **use it to authenticate against the ntfy server, e.g. when you publish or
subscribe to topics**. To learn how, check out [authenticate via access tokens](publish.md#access-tokens).

#### Tokens via the config
Access tokens can be pre-provisioned in the `server.yml` configuration file using the `auth-tokens` config option.
This is useful for automated setups, Docker environments, or when you want to define tokens declaratively.

The `auth-tokens` option is a list of access tokens that are automatically created/updated when the server starts.
When entries are removed, they are deleted from the database. Each entry is defined in the format `<username>:<token>[:<label>]`.

The `<username>` must be an existing, provisioned user, as defined in the `auth-users` section (see [users via the config](#users-via-the-config)).
The `<token>` is a valid access token, which must start with `tk_` and be 32 characters long (including the prefix). You can generate
random tokens using the `ntfy token generate` command. The optional `<label>` is a human-readable label for the token, 
which can be used to identify it later.

Once configured, these tokens can be used to authenticate API requests just like tokens created via the CLI.
For usage examples, see [authenticate via access tokens](publish.md#access-tokens).

Here's an example:

=== "Declarative tokens in /etc/ntfy/server.yml"
    ``` yaml
    auth-file: "/var/lib/ntfy/user.db"
    auth-users:
      - "phil:$2a$10$YLiO8U21sX1uhZamTLJXHuxgVC0Z/GKISibrKCLohPgtG7yIxSk4C:admin"
      - "backup-service:$2a$10$NKbrNb7HPMjtQXWJ0f1pouw03LDLT/WzlO9VAv44x84bRCkh19h6m:user"
    auth-tokens:
      - "phil:tk_3gd7d2yftt4b8ixyfe9mnmro88o76"
      - "backup-service:tk_f099we8uzj7xi5qshzajwp6jffvkz:Backup script"
    ```

=== "Declarative tokens via env variables"
    ```
    # Comma-separated list
    NTFY_AUTH_FILE='/var/lib/ntfy/user.db'
    NTFY_AUTH_USERS='phil:$2a$10$YLiO8U21sX1uhZamTLJXHuxgVC0Z/GKISibrKCLohPgtG7yIxSk4C:admin,ben:$2a$10$NKbrNb7HPMjtQXWJ0f1pouw03LDLT/WzlO9VAv44x84bRCkh19h6m:user'
    NTFY_AUTH_TOKENS='phil:tk_3gd7d2yftt4b8ixyfe9mnmro88o76,backup-service:tk_f099we8uzj7xi5qshzajwp6jffvkz:Backup script'
    ```

In this example, the `auth-users` section defines two users, `phil` and `backup-service`. The `auth-tokens` section
defines access tokens for these users. `phil` has a token `tk_3gd7d2yftt4b8ixyfe9mnmro88o76`, while `backup-service`
has a token `tk_f099we8uzj7xi5qshzajwp6jffvkz` with the label "Backup script".

### Example: Private instance
The easiest way to configure a private instance is to set `auth-default-access` to `deny-all` in the `server.yml`,
and to configure users in the `auth-users` section (see [users via the config](#users-via-the-config)), 
access control entries in the `auth-access` section (see [ACL entries via the config](#acl-entries-via-the-config)),
and access tokens in the `auth-tokens` section (see [access tokens via the config](#tokens-via-the-config)).

Here's an example that defines a single admin user `phil` with the password `mypass`, and a regular user `backup-script`
with the password `backup-script`. The admin user has full access to all topics, while regular user can only
access the `backups` topic with read-write permissions. The `auth-default-access` is set to `deny-all`, which means
that all other users and anonymous access are denied by default.

=== "Config via /etc/ntfy/server.yml"
    ``` yaml
    auth-file: "/var/lib/ntfy/user.db"
    auth-default-access: "deny-all"
    auth-users:
      - "phil:$2a$10$YLiO8U21sX1uhZamTLJXHuxgVC0Z/GKISibrKCLohPgtG7yIxSk4C:admin"
      - "backup-script:$2a$10$/ehiQt.w7lhTmHXq.RNsOOkIwiPPeWFIzWYO3DRxNixnWKLX8.uj.:user"
    auth-access:
      - "backup-service:backups:rw"
    auth-tokens:
      - "phil:tk_3gd7d2yftt4b8ixyfe9mnmro88o76:My personal token"
    ```

=== "Config via env variables"
    ``` yaml
    NTFY_AUTH_FILE='/var/lib/ntfy/user.db'
    NTFY_AUTH_DEFAULT_ACCESS='deny-all'
    NTFY_AUTH_USERS='phil:$2a$10$YLiO8U21sX1uhZamTLJXHuxgVC0Z/GKISibrKCLohPgtG7yIxSk4C:admin,backup-script:$2a$10$/ehiQt.w7lhTmHXq.RNsOOkIwiPPeWFIzWYO3DRxNixnWKLX8.uj.:user'
    NTFY_AUTH_ACCESS='backup-service:backups:rw'
    NTFY_AUTH_TOKENS='phil:tk_3gd7d2yftt4b8ixyfe9mnmro88o76:My personal token'
    ```

Once you've done that, you can publish and subscribe using [Basic Auth](https://en.wikipedia.org/wiki/Basic_access_authentication) 
with the given username/password. Be sure to use HTTPS to avoid eavesdropping and exposing your password. 

Here's a simple example (using the credentials of the `phil` user):

=== "Command line (curl)"
    ```
    curl \
        -u phil:mypass \
        -d "Look ma, with auth" \
        https://ntfy.example.com/mysecrets
    ```

=== "ntfy CLI"
    ```
    ntfy publish \
        -u phil:mypass \
        ntfy.example.com/mysecrets \
        "Look ma, with auth"
    ```

=== "HTTP"
    ``` http
    POST /mysecrets HTTP/1.1
    Host: ntfy.example.com
    Authorization: Basic cGhpbDpteXBhc3M=

    Look ma, with auth
    ```

=== "JavaScript"
    ``` javascript
    fetch('https://ntfy.example.com/mysecrets', {
        method: 'POST', // PUT works too
        body: 'Look ma, with auth',
        headers: {
            'Authorization': 'Basic cGhpbDpteXBhc3M='
        }
    })
    ```

=== "Go"
    ``` go
    req, _ := http.NewRequest("POST", "https://ntfy.example.com/mysecrets",
        strings.NewReader("Look ma, with auth"))
    req.Header.Set("Authorization", "Basic cGhpbDpteXBhc3M=")
    http.DefaultClient.Do(req)
    ```

=== "Python"
    ``` python
    requests.post("https://ntfy.example.com/mysecrets",
        data="Look ma, with auth",
        headers={
            "Authorization": "Basic cGhpbDpteXBhc3M="
        })
    ```

=== "PHP"
    ``` php-inline
    file_get_contents('https://ntfy.example.com/mysecrets', false, stream_context_create([
        'http' => [
            'method' => 'POST', // PUT also works
            'header' => 
                'Content-Type: text/plain\r\n' .
                'Authorization: Basic cGhpbDpteXBhc3M=',
            'content' => 'Look ma, with auth'
        ]
    ]));
    ```

### Example: UnifiedPush
[UnifiedPush](https://unifiedpush.org) requires that the [application server](https://unifiedpush.org/developers/spec/definitions/#application-server) (e.g. Synapse, Fediverse Server, …) 
has anonymous write access to the [topic](https://unifiedpush.org/developers/spec/definitions/#endpoint) used for push messages. 
The topic names used by UnifiedPush all start with the `up*` prefix. Please refer to the 
**[UnifiedPush documentation](https://unifiedpush.org/users/distributors/ntfy/#limit-access-to-some-users-acl)** for more details.

To enable support for UnifiedPush for private servers (i.e. `auth-default-access: "deny-all"`), you should either 
allow anonymous write access for the entire prefix or explicitly per topic:

=== "Prefix"
    ```
    $ ntfy access '*' 'up*' write-only
    ```

=== "Explicitly"
    ```
    $ ntfy access '*' upYzMtZGZiYTY5 write-only
    ```

## E-mail notifications
To allow forwarding messages via e-mail, you can configure an **SMTP server for outgoing messages**. Once configured, 
you can set the `X-Email` header to [send messages via e-mail](publish.md#e-mail-notifications) (e.g. 
`curl -d "hi there" -H "X-Email: phil@example.com" ntfy.sh/mytopic`).

As of today, only SMTP servers with PLAIN auth and STARTLS are supported. To enable e-mail sending, you must set the 
following settings:

* `base-url` is the root URL for the ntfy server; this is needed for e-mail footer
* `smtp-sender-addr` is the hostname:port of the SMTP server
* `smtp-sender-user` and `smtp-sender-pass` are the username and password of the SMTP user
* `smtp-sender-from` is the e-mail address of the sender

Here's an example config using [Amazon SES](https://aws.amazon.com/ses/) for outgoing mail (this is how it is 
configured for `ntfy.sh`):

=== "/etc/ntfy/server.yml"
    ``` yaml
    base-url: "https://ntfy.sh"
    smtp-sender-addr: "email-smtp.us-east-2.amazonaws.com:587"
    smtp-sender-user: "AKIDEADBEEFAFFE12345"
    smtp-sender-pass: "Abd13Kf+sfAk2DzifjafldkThisIsNotARealKeyOMG."
    smtp-sender-from: "ntfy@ntfy.sh"
    ```

Please also refer to the [rate limiting](#rate-limiting) settings below, specifically `visitor-email-limit-burst` 
and `visitor-email-limit-burst`. Setting these conservatively is necessary to avoid abuse.

## E-mail publishing
To allow publishing messages via e-mail, ntfy can run a lightweight **SMTP server for incoming messages**. Once configured, 
users can [send emails to a topic e-mail address](publish.md#e-mail-publishing) (e.g. `mytopic@ntfy.sh` or 
`myprefix-mytopic@ntfy.sh`) to publish messages to a topic. This is useful for e-mail based integrations such as for 
statuspage.io (though these days most services also support webhooks and HTTP calls).

To configure the SMTP server, you must at least set `smtp-server-listen` and `smtp-server-domain`:

* `smtp-server-listen` defines the IP address and port the SMTP server will listen on, e.g. `:25` or `1.2.3.4:25`
* `smtp-server-domain` is the e-mail domain, e.g. `ntfy.sh` (must be identical to MX record, see below)
* `smtp-server-addr-prefix` is an optional prefix for the e-mail addresses to prevent spam. If set to `ntfy-`, for instance,
  only e-mails to `ntfy-$topic@ntfy.sh` will be accepted. If this is not set, all emails to `$topic@ntfy.sh` will be
  accepted (which may obviously be a spam problem).

Here's an example config (this is how it is configured for `ntfy.sh`):

=== "/etc/ntfy/server.yml"
    ``` yaml
    smtp-server-listen: ":25"
    smtp-server-domain: "ntfy.sh"
    smtp-server-addr-prefix: "ntfy-"
    ```

In addition to configuring the ntfy server, you have to create two DNS records (an [MX record](https://en.wikipedia.org/wiki/MX_record) 
and a corresponding A record), so incoming mail will find its way to your server. Here's an example of how `ntfy.sh` is 
configured (in [Amazon Route 53](https://aws.amazon.com/route53/)):

<figure markdown>
  ![DNS records for incoming mail](static/img/screenshot-email-publishing-dns.png){ width=600 }
  <figcaption>DNS records for incoming mail</figcaption>
</figure>

You can check if everything is working correctly by sending an email as raw SMTP via `nc`. Create a text file, e.g. 
`email.txt`

```
EHLO example.com
MAIL FROM: phil@example.com
RCPT TO: ntfy-mytopic@ntfy.sh
DATA
Subject: Email for you
Content-Type: text/plain; charset="UTF-8"

Hello from 🇩🇪
.
```

And then send the mail via `nc` like this. If you see any lines starting with `451`, those are errors from the 
ntfy server. Read them carefully.

```
$ cat email.txt | nc -N ntfy.sh 25
220 ntfy.sh ESMTP Service Ready
250-Hello example.com
...
250 2.0.0 Roger, accepting mail from <phil@example.com>
250 2.0.0 I'll make sure <ntfy-mytopic@ntfy.sh> gets this
```

As for the DNS setup, be sure to verify that `dig MX` and `dig A` are returning results similar to this:

```
$ dig MX ntfy.sh +short 
10 mx1.ntfy.sh.
$ dig A mx1.ntfy.sh +short 
3.139.215.220
```

### Local-only email
If you want to send emails from an internal service on the same network as your ntfy instance, you do not need to
worry about DNS records at all. Define a port for the SMTP server and pick an SMTP server domain (can be
anything).

=== "/etc/ntfy/server.yml"
    ``` yaml
    smtp-server-listen: ":25"
    smtp-server-domain: "example.com"
    smtp-server-addr-prefix: "ntfy-"  # optional
    ```

Then, in the email settings of your internal service, set the SMTP server address to the IP address of your
ntfy instance. Set the port to the value you defined in `smtp-server-listen`. Leave any username and password
fields empty. In the "From" address, pick anything (e.g., "alerts@ntfy.sh"); the value doesn't matter.
In the "To" address, put in an email address that follows this pattern: `[topic]@[smtp-server-domain]` (or
`[smtp-server-addr-prefix][topic]@[smtp-server-domain]` if you set `smtp-server-addr-prefix`).

So if you used `example.com` as the SMTP server domain, and you want to send a message to the `email-alerts`
topic, set the "To" address to `email-alerts@example.com`. If the topic has access restrictions, you will need
to include an access token in the "To" address, such as `email-alerts+tk_AbC123dEf456@example.com`.

If the internal service lets you use define an email "Subject", it will become the title of the notification.
The body of the email will become the message of the notification.

## Behind a proxy (TLS, etc.)
!!! warning
    If you are running ntfy behind a proxy, you must set the `behind-proxy` flag. Otherwise, all visitors are
    [rate limited](#rate-limiting) as if they are one.

It may be desirable to run ntfy behind a proxy (e.g. nginx, HAproxy or Apache), so you can provide TLS certificates 
using Let's Encrypt using certbot, or simply because you'd like to share the ports (80/443) with other services. 
Whatever your reasons may be, there are a few things to consider. 

### IP-based rate limiting
If you are running ntfy behind a proxy, you should set the `behind-proxy` flag. This will instruct the 
[rate limiting](#rate-limiting) logic to use the header configured in `proxy-forwarded-header` (default is `X-Forwarded-For`)
as the primary identifier for a visitor, as opposed to the remote IP address. 

If the `behind-proxy` flag is not set, all visitors will be counted as one, because from the perspective of the
ntfy server, they all share the proxy's IP address.

Relevant flags to consider:

* `behind-proxy` makes it so that the real visitor IP address is extracted from the header defined in `proxy-forwarded-header`.
  Without this, the remote address of the incoming connection is used (default: `false`).
* `proxy-forwarded-header` is the header to use to identify visitors (default: `X-Forwarded-For`). It may be a single IP address (e.g. `1.2.3.4`),
  a comma-separated list of IP addresses (e.g. `1.2.3.4, 5.6.7.8`), or an [RFC 7239](https://datatracker.ietf.org/doc/html/rfc7239)-style
 header (e.g. `for=1.2.3.4;by=proxy.example.com, for=5.6.7.8`).
* `proxy-trusted-hosts` is a comma-separated list of IP addresses, hosts or CIDRs that are removed from the forwarded header 
  to determine the real IP address. This is only useful if there are multiple proxies involved that add themselves to
  the forwarded header (default: empty).
* `visitor-prefix-bits-ipv4` is the number of bits of the IPv4 address to use for rate limiting (default is `32`, which is the entire
  IP address). In IPv4 environments, by default, a visitor's **full IPv4 address** is used as-is for rate limiting. This means that
  if someone publishes messages from multiple IP addresses, they will be counted as separate visitors. You can adjust this by setting the `visitor-prefix-bits-ipv4` config option. To group visitors in a /24 subnet and count them as one, for instance,
  set it to `24`. In that case, `1.2.3.4` and `1.2.3.99` are treated as the same visitor.
* `visitor-prefix-bits-ipv6` is the number of bits of the IPv6 address to use for rate limiting (default is `64`, which is a /64 subnet). 
  In IPv6 environments, by default, a visitor's IP address is **truncated to the /64 subnet**, meaning that `2001:db8:25:86:1::1` and 
  `2001:db8:25:86:2::1` are treated as the same visitor. Use the `visitor-prefix-bits-ipv6` config option to adjust this behavior.
  See [IPv6 considerations](#ipv6-considerations) for more details.

=== "/etc/ntfy/server.yml (behind a proxy)"
    ``` yaml
    # Tell ntfy to use "X-Forwarded-For" header to identify visitors for rate limiting
    #
    # Example: If "X-Forwarded-For: 9.9.9.9, 1.2.3.4" is set, 
    #          the visitor IP will be 1.2.3.4 (right-most address).
    #
    behind-proxy: true
    ```

=== "/etc/ntfy/server.yml (X-Client-IP header)"
    ``` yaml
    # Tell ntfy to use "X-Client-IP" header to identify visitors for rate limiting
    #
    # Example: If "X-Client-IP: 9.9.9.9" is set, 
    #          the visitor IP will be 9.9.9.9.
    #
    behind-proxy: true
    proxy-forwarded-header: "X-Client-IP"
    ```

=== "/etc/ntfy/server.yml (Forwarded header)"
    ``` yaml
    # Tell ntfy to use "Forwarded" header (RFC 7239) to identify visitors for rate limiting
    #
    # Example: If "Forwarded: for=1.2.3.4;by=proxy.example.com, for=9.9.9.9" is set, 
    #          the visitor IP will be 9.9.9.9.
    #
    behind-proxy: true
    proxy-forwarded-header: "Forwarded"
    ```

=== "/etc/ntfy/server.yml (multiple proxies)"
    ``` yaml
    # Tell ntfy to use "X-Forwarded-For" header to identify visitors for rate limiting,
    # and to strip the IP addresses of the proxies 1.2.3.4 and 1.2.3.5
    #
    # Example: If "X-Forwarded-For: 9.9.9.9, 1.2.3.4" is set, 
    #          the visitor IP will be 9.9.9.9 (right-most unknown address).
    #
    behind-proxy: true
    proxy-trusted-hosts: "1.2.3.0/24, 1.2.2.2, 2001:db8::/64"
    ```

=== "/etc/ntfy/server.yml (adjusted IPv4/IPv6 prefixes proxies)"
    ``` yaml
    # Tell ntfy to treat visitors as being in a /24 subnet (IPv4) or /48 subnet (IPv6)
    # as one visitor, so that they are counted as one for rate limiting.
    #
    # Example 1: If 1.2.3.4 and 1.2.3.5 publish a message, the visitor 1.2.3.0 will have
    #            used 2 messages.
    # Example 2: If 2001:db8:2500:1::1 and 2001:db8:2500:2::1 publish a message, the visitor
    #            2001:db8:2500:: will have used 2 messages.
    #
    visitor-prefix-bits-ipv4: 24
    visitor-prefix-bits-ipv6: 48
    ```

### TLS/SSL
ntfy supports HTTPS/TLS by setting the `listen-https` [config option](#config-options). However, if you 
are behind a proxy, it is recommended that TLS/SSL termination is done by the proxy itself (see below).

I highly recommend using [certbot](https://certbot.eff.org/). I use it with the [dns-route53 plugin](https://certbot-dns-route53.readthedocs.io/en/stable/), 
which lets you use [AWS Route 53](https://aws.amazon.com/route53/) as the challenge. That's much easier than using the
HTTP challenge. I've found [this guide](https://nandovieira.com/using-lets-encrypt-in-development-with-nginx-and-aws-route53) to
be incredibly helpful.

### nginx/Apache2/caddy
For your convenience, here's a working config that'll help configure things behind a proxy. Be sure to **enable WebSockets**
by forwarding the `Connection` and `Upgrade` headers accordingly. 

In this example, ntfy runs on `:2586` and we proxy traffic to it. We also redirect HTTP to HTTPS for GET requests against a topic
or the root domain:

=== "nginx (convenient)"
    ```
    # /etc/nginx/sites-*/ntfy
    #
    # This config allows insecure HTTP POST/PUT requests against topics to allow a short curl syntax (without -L
    # and "https://" prefix). It also disables output buffering, which has worked well for the ntfy.sh server.
    #
    # This is pretty much how ntfy.sh is configured. To see the exact configuration,
    # see https://github.com/binwiederhier/ntfy-ansible/

    server {
      listen 80;
      server_name ntfy.sh;

      location / {
        # Redirect HTTP to HTTPS, but only for GET topic addresses, since we want 
        # it to work with curl without the annoying https:// prefix
        set $redirect_https "";
        if ($request_method = GET) {
          set $redirect_https "yes";
        }
        if ($request_uri ~* "^/([-_a-z0-9]{0,64}$|docs/|static/)") {
          set $redirect_https "${redirect_https}yes";
        }
        if ($redirect_https = "yesyes") {
          return 302 https://$http_host$request_uri$is_args$query_string;
        }

        proxy_pass http://127.0.0.1:2586;
        proxy_http_version 1.1;
    
        proxy_buffering off;
        proxy_request_buffering off;
        proxy_redirect off;
     
        proxy_set_header Host $http_host;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    
        proxy_connect_timeout 3m;
        proxy_send_timeout 3m;
        proxy_read_timeout 3m;

        client_max_body_size 0; # Stream request body to backend
      }
    }
    
    server {
      listen 443 ssl http2;
      server_name ntfy.sh;
    
      # See https://ssl-config.mozilla.org/#server=nginx&version=1.18.0&config=intermediate&openssl=1.1.1k&hsts=false&ocsp=false&guideline=5.6
      ssl_session_timeout 1d;
      ssl_session_cache shared:MozSSL:10m; # about 40000 sessions
      ssl_session_tickets off;
      ssl_protocols TLSv1.2 TLSv1.3;
      ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
      ssl_prefer_server_ciphers off;
 
      ssl_certificate /etc/letsencrypt/live/ntfy.sh/fullchain.pem;
      ssl_certificate_key /etc/letsencrypt/live/ntfy.sh/privkey.pem;
    
      location / {
        proxy_pass http://127.0.0.1:2586;
        proxy_http_version 1.1;

        proxy_buffering off;
        proxy_request_buffering off;
        proxy_redirect off;
     
        proxy_set_header Host $http_host;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    
        proxy_connect_timeout 3m;
        proxy_send_timeout 3m;
        proxy_read_timeout 3m;
        
        client_max_body_size 0; # Stream request body to backend
      }
    }
    ```

=== "nginx (more secure)"
    ```
    # /etc/nginx/sites-*/ntfy
    #
    # This config requires the use of the -L flag in curl to redirect to HTTPS, and it keeps nginx output buffering
    # enabled. While recommended, I have had issues with that in the past.
    
    server {
      listen 80;
      server_name ntfy.sh;

      location / {
        return 302 https://$http_host$request_uri$is_args$query_string;

        proxy_pass http://127.0.0.1:2586;
        proxy_http_version 1.1;

        proxy_set_header Host $http_host;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        proxy_connect_timeout 3m;
        proxy_send_timeout 3m;
        proxy_read_timeout 3m;

        client_max_body_size 0; # Stream request body to backend
      }
    }
    
    server {
      listen 443 ssl http2;
      server_name ntfy.sh;
    
      # See https://ssl-config.mozilla.org/#server=nginx&version=1.18.0&config=intermediate&openssl=1.1.1k&hsts=false&ocsp=false&guideline=5.6
      ssl_session_timeout 1d;
      ssl_session_cache shared:MozSSL:10m; # about 40000 sessions
      ssl_session_tickets off;
      ssl_protocols TLSv1.2 TLSv1.3;
      ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
      ssl_prefer_server_ciphers off;
    
      ssl_certificate /etc/letsencrypt/live/ntfy.sh/fullchain.pem;
      ssl_certificate_key /etc/letsencrypt/live/ntfy.sh/privkey.pem;
    
      location / {
        proxy_pass http://127.0.0.1:2586;
        proxy_http_version 1.1;

        proxy_set_header Host $http_host;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        proxy_connect_timeout 3m;
        proxy_send_timeout 3m;
        proxy_read_timeout 3m;

        client_max_body_size 0; # Stream request body to backend
      }
    }
    ```

=== "Apache2"
    ```
    # /etc/apache2/sites-*/ntfy.conf

    <VirtualHost *:80>
        ServerName ntfy.sh

        # Proxy connections to ntfy (requires "a2enmod proxy proxy_http")
        ProxyPass / http://127.0.0.1:2586/ upgrade=websocket
        ProxyPassReverse / http://127.0.0.1:2586/

        SetEnv proxy-nokeepalive 1
        SetEnv proxy-sendchunked 1

        # Higher than the max message size of 4096 bytes
        LimitRequestBody 102400
        
        # Redirect HTTP to HTTPS, but only for GET topic addresses, since we want 
        # it to work with curl without the annoying https:// prefix (requires "a2enmod alias")
        <If "%{REQUEST_METHOD} == 'GET'">
            RedirectMatch permanent "^/([-_A-Za-z0-9]{0,64})$" "https://%{SERVER_NAME}/$1"
        </If>

    </VirtualHost>
    
    <VirtualHost *:443>
        ServerName ntfy.sh
        
        SSLEngine on
        SSLCertificateFile /etc/letsencrypt/live/ntfy.sh/fullchain.pem
        SSLCertificateKeyFile /etc/letsencrypt/live/ntfy.sh/privkey.pem
        Include /etc/letsencrypt/options-ssl-apache.conf

        # Proxy connections to ntfy (requires "a2enmod proxy proxy_http")
        ProxyPass / http://127.0.0.1:2586/ upgrade=websocket
        ProxyPassReverse / http://127.0.0.1:2586/

        SetEnv proxy-nokeepalive 1
        SetEnv proxy-sendchunked 1

        # Higher than the max message size of 4096 bytes 
        LimitRequestBody 102400
	
    </VirtualHost>
    ```

=== "caddy"
    ```
    # Note that this config is most certainly incomplete. Please help out and let me know what's missing
    # via Discord/Matrix or in a GitHub issue.
    # Note: Caddy automatically handles both HTTP and WebSockets with reverse_proxy 

    ntfy.sh, http://nfty.sh {
        reverse_proxy 127.0.0.1:2586

        # Redirect HTTP to HTTPS, but only for GET topic addresses, since we want
        # it to work with curl without the annoying https:// prefix
        @httpget {
            protocol http
            method GET
            path_regexp ^/([-_a-z0-9]{0,64}$|docs/|static/)
        }
        redir @httpget https://{host}{uri}
    }
    ```

## Firebase (FCM)
!!! info
    Using Firebase is **optional** and only works if you modify and [build your own Android .apk](develop.md#android-app).
    For a self-hosted instance, it's easier to just not bother with FCM.

[Firebase Cloud Messaging (FCM)](https://firebase.google.com/docs/cloud-messaging) is the Google approved way to send
push messages to Android devices. FCM is the only method that an Android app can receive messages without having to run a
[foreground service](https://developer.android.com/guide/components/foreground-services).

For the main host [ntfy.sh](https://ntfy.sh), the [ntfy Android app](subscribe/phone.md) uses Firebase to send messages
to the device. For other hosts, instant delivery is used and FCM is not involved.

To configure FCM for your self-hosted instance of the ntfy server, follow these steps:

1. Sign up for a [Firebase account](https://console.firebase.google.com/)
2. Create a Firebase app and download the key file (e.g. `myapp-firebase-adminsdk-...json`)
3. Place the key file in `/etc/ntfy`, set the `firebase-key-file` in `server.yml` accordingly and restart the ntfy server
4. Build your own Android .apk following [these instructions](develop.md#android-app)

Example:
```
# If set, also publish messages to a Firebase Cloud Messaging (FCM) topic for your app.
# This is optional and only required to support Android apps (which don't allow background services anymore).
#
firebase-key-file: "/etc/ntfy/ntfy-sh-firebase-adminsdk-ahnce-9f4d6f14b5.json"
```

## iOS instant notifications
Unlike Android, iOS heavily restricts background processing, which sadly makes it impossible to implement instant 
push notifications without a central server. 

To still support instant notifications on iOS through your self-hosted ntfy server, you have to forward so called `poll_request` 
messages to the main ntfy.sh server (or any upstream server that's APNS/Firebase connected, if you build your own iOS app),
which will then forward it to Firebase/APNS.

To configure it, simply set `upstream-base-url` like so:

``` yaml
upstream-base-url: "https://ntfy.sh"
upstream-access-token: "..." # optional, only if rate limits exceeded, or upstream server protected
```

If set, all incoming messages will publish a poll request to the configured upstream server, containing
the message ID of the original message, instructing the iOS app to poll this server for the actual message contents.

If `upstream-base-url` is not set, notifications will still eventually get to your device, but delivery can take hours,
depending on the state of the phone. If you are using your phone, it shouldn't take more than 20-30 minutes though.

In case you're curious, here's an example of the entire flow: 

- In the iOS app, you subscribe to `https://ntfy.example.com/mytopic`
- The app subscribes to the Firebase topic `6de73be8dfb7d69e...` (the SHA256 of the topic URL)
- When you publish a message to `https://ntfy.example.com/mytopic`, your ntfy server will publish a 
  poll request to `https://ntfy.sh/6de73be8dfb7d69e...`. The request from your server to the upstream server 
  contains only the message ID (in the `X-Poll-ID` header), and the SHA256 checksum of the topic URL (as upstream topic).
- The ntfy.sh server publishes the poll request message to Firebase, which forwards it to APNS, which forwards it to your iOS device
- Your iOS device receives the poll request, and fetches the actual message from your server, and then displays it

Here's an example of what the self-hosted server forwards to the upstream server. The request is equivalent to this curl:

```
curl -X POST -H "X-Poll-ID: s4PdJozxM8na" https://ntfy.sh/6de73be8dfb7d69e32fb2c00c23fe7adbd8b5504406e3068c273aa24cef4055b
{"id":"4HsClFEuCIcs","time":1654087955,"event":"poll_request","topic":"6de73be8dfb7d69e32fb2c00c23fe7adbd8b5504406e3068c273aa24cef4055b","message":"New message","poll_id":"s4PdJozxM8na"}
```

Note that the self-hosted server literally sends the message `New message` for every message, even if your message 
may be `Some other message`. This is so that if iOS cannot talk to the self-hosted server (in time, or at all), 
it'll show `New message` as a popup.

## Web Push
[Web Push](https://developer.mozilla.org/en-US/docs/Web/API/Push_API) ([RFC8030](https://datatracker.ietf.org/doc/html/rfc8030))
allows ntfy to receive push notifications, even when the ntfy web app (or even the browser, depending on the platform) is closed. 
When enabled, the user can enable **background notifications** for their topics in the web app under Settings. Once enabled by the
user, ntfy will forward published messages to the push endpoint (browser-provided, e.g. fcm.googleapis.com), which will then
forward it to the browser.

To configure Web Push, you need to generate and configure a [VAPID](https://datatracker.ietf.org/doc/html/draft-thomson-webpush-vapid) keypair (via `ntfy webpush keys`),
a database to keep track of the browser's subscriptions, and an admin email address (you):

- `web-push-public-key` is the generated VAPID public key, e.g. AA1234BBCCddvveekaabcdfqwertyuiopasdfghjklzxcvbnm1234567890
- `web-push-private-key` is the generated VAPID private key, e.g. AA2BB1234567890abcdefzxcvbnm1234567890
- `web-push-file` is a database file to keep track of browser subscription endpoints, e.g. `/var/cache/ntfy/webpush.db`
- `web-push-email-address` is the admin email address send to the push provider, e.g. `sysadmin@example.com`
- `web-push-startup-queries` is an optional list of queries to run on startup`
- `web-push-expiry-warning-duration` defines the duration after which unused subscriptions are sent a warning (default is `55d`)
- `web-push-expiry-duration` defines the duration after which unused subscriptions will expire (default is `60d`)

Limitations:

- Like foreground browser notifications, background push notifications require the web app to be served over HTTPS. A _valid_
  certificate is required, as service workers will not run on origins with untrusted certificates.

- Web Push is only supported for the same server. You cannot use subscribe to web push on a topic on another server. This
  is due to a limitation of the Push API, which doesn't allow multiple push servers for the same origin.

To configure VAPID keys, first generate them:

```sh
$ ntfy webpush keys
Web Push keys generated.
...
```

Then copy the generated values into your `server.yml` or use the corresponding environment variables or command line arguments:

```yaml
web-push-public-key: AA1234BBCCddvveekaabcdfqwertyuiopasdfghjklzxcvbnm1234567890
web-push-private-key: AA2BB1234567890abcdefzxcvbnm1234567890
web-push-file: /var/cache/ntfy/webpush.db
web-push-email-address: sysadmin@example.com
```

The `web-push-file` is used to store the push subscriptions. Unused subscriptions will send out a warning after 55 days,
and will automatically expire after 60 days (default). If the gateway returns an error (e.g. 410 Gone when a user has unsubscribed),
subscriptions are also removed automatically.

The web app refreshes subscriptions on start and regularly on an interval, but this file should be persisted across restarts. If the subscription
file is deleted or lost, any web apps that aren't open will not receive new web push notifications until you open then.

Changing your public/private keypair is **not recommended**. Browsers only allow one server identity (public key) per origin, and
if you change them the clients will not be able to subscribe via web push until the user manually clears the notification permission.

## Tiers
ntfy supports associating users to pre-defined tiers. Tiers can be used to grant users higher limits, such as 
daily message limits, attachment size, or make it possible for users to reserve topics. If [payments are enabled](#payments),
tiers can be paid or unpaid, and users can upgrade/downgrade between them. If payments are disabled, then the only way
to switch between tiers is with the `ntfy user change-tier` command (see [users and roles](#users-and-roles)).

By default, **newly created users have no tier**, and all usage limits are read from the `server.yml` config file.
Once a user is associated with a tier, some limits are overridden based on the tier.

The `ntfy tier` command can be used to manage all available tiers. By default, there are no pre-defined tiers.

**Example commands** (type `ntfy token --help` or `ntfy token COMMAND --help` for more details):
```
ntfy tier add pro                     # Add tier with code "pro", using the defaults
ntfy tier change --name="Pro" pro     # Update the name of an existing tier
ntfy tier del starter                 # Delete an existing tier
ntfy user change-tier phil pro        # Switch user "phil" to tier "pro"
```

**Creating a tier (full example):**
```
ntfy tier add \
  --name="Pro" \
  --message-limit=10000 \
  --message-expiry-duration=24h \
  --email-limit=50 \
  --call-limit=10 \
  --reservation-limit=10 \
  --attachment-file-size-limit=100M \
  --attachment-total-size-limit=1G \
  --attachment-expiry-duration=12h \
  --attachment-bandwidth-limit=5G \
  --stripe-price-id=price_123456 \
  pro
```

## Payments
ntfy supports paid [tiers](#tiers) via [Stripe](https://stripe.com/) as a payment provider. If payments are enabled,
users can register, login and switch plans in the web app. The web app will behave slightly differently if payments 
are enabled (e.g. showing an upgrade banner, or "ntfy Pro" tags).

!!! info
    The ntfy payments integration is very tailored to ntfy.sh and Stripe. I do not intend to support arbitrary use
    cases.

To enable payments, sign up with [Stripe](https://stripe.com/), set the `stripe-secret-key` and `stripe-webhook-key`
config options: 

* `stripe-secret-key` is the key used for the Stripe API communication. Setting this values
   enables payments in the ntfy web app (e.g. Upgrade dialog). See [API keys](https://dashboard.stripe.com/apikeys).
* `stripe-webhook-key` is the key required to validate the authenticity of incoming webhooks from Stripe.
   Webhooks are essential to keep the local database in sync with the payment provider. See [Webhooks](https://dashboard.stripe.com/webhooks).
* `billing-contact` is an email address or website displayed in the "Upgrade tier" dialog to let people reach
   out with billing questions. If unset, nothing will be displayed.

In addition to setting these two options, you also need to define a [Stripe webhook](https://dashboard.stripe.com/webhooks)
for the `customer.subscription.updated` and `customer.subscription.deleted` event, which points 
to `https://ntfy.example.com/v1/account/billing/webhook`.

Here's an example:

``` yaml
stripe-secret-key: "sk_test_ZmhzZGtmbGhkc2tqZmhzYcO2a2hmbGtnaHNkbGtnaGRsc2hnbG"
stripe-webhook-key: "whsec_ZnNkZnNIRExBSFNES0hBRFNmaHNka2ZsaGR"
billing-contact: "phil@example.com"
```

## Phone calls
ntfy supports phone calls via [Twilio](https://www.twilio.com/) as a call provider. If phone calls are enabled,
users can verify and add a phone number, and then receive phone calls when publishing a message using the `X-Call` header.
See [publishing page](publish.md#phone-calls) for more details.

To enable Twilio integration, sign up with [Twilio](https://www.twilio.com/), purchase a phone number (Toll free numbers
are the easiest), and then configure the following options:

* `twilio-account` is the Twilio account SID, e.g. AC12345beefbeef67890beefbeef122586
* `twilio-auth-token` is the Twilio auth token, e.g. affebeef258625862586258625862586
* `twilio-phone-number` is the outgoing phone number you purchased, e.g. +18775132586 
* `twilio-verify-service` is the Twilio Verify service SID, e.g. VA12345beefbeef67890beefbeef122586

After you have configured phone calls, create a [tier](#tiers) with a call limit (e.g. `ntfy tier create --call-limit=10 ...`),
and then assign it to a user. Users may then use the `X-Call` header to receive a phone call when publishing a message.

## Message limits
There are a few message limits that you can configure:

* `message-size-limit` defines the max size of a message body. Please note message sizes >4K are **not recommended,
   and largely untested**. The Android/iOS and other clients may not work, or work properly. If FCM and/or APNS is used,
   the limit should stay 4K, because their limits are around that size. If you increase this size limit regardless, 
   FCM and APNS will NOT work for large messages.
* `message-delay-limit` defines the max delay of a message when using the "Delay" header and [scheduled delivery](publish.md#scheduled-delivery).

## Rate limiting
!!! info
    Be aware that if you are running ntfy behind a proxy, you must set the `behind-proxy` flag. 
    Otherwise, all visitors are rate limited as if they are one.

By default, ntfy runs without authentication, so it is vitally important that we protect the server from abuse or overload.
There are various limits and rate limits in place that you can use to configure the server:

* **Global limit**: A global limit applies across all visitors (IPs, clients, users)
* **Visitor limit**: A visitor limit only applies to a certain visitor. A **visitor** is identified by its IP address 
  (or the `X-Forwarded-For` header if `behind-proxy` is set). All config options that start with the word `visitor` apply 
  only on a per-visitor basis.

During normal usage, you shouldn't encounter these limits at all, and even if you burst a few requests or emails
(e.g. when you reconnect after a connection drop), it shouldn't have any effect.

### General limits
Let's do the easy limits first:

* `global-topic-limit` defines the total number of topics before the server rejects new topics. It defaults to 15,000.
* `visitor-subscription-limit` is the number of subscriptions (open connections) per visitor. This value defaults to 30.

### Request limits
In addition to the limits above, there is a requests/second limit per visitor for all sensitive GET/PUT/POST requests.
This limit uses a [token bucket](https://en.wikipedia.org/wiki/Token_bucket) (using Go's [rate package](https://pkg.go.dev/golang.org/x/time/rate)):

Each visitor has a bucket of 60 requests they can fire against the server (defined by `visitor-request-limit-burst`). 
After the 60, new requests will encounter a `429 Too Many Requests` response. The visitor request bucket is refilled at a rate of one
request every 5s (defined by `visitor-request-limit-replenish`)

* `visitor-request-limit-burst` is the initial bucket of requests each visitor has. This defaults to 60.
* `visitor-request-limit-replenish` is the rate at which the bucket is refilled (one request per x). Defaults to 5s.
* `visitor-request-limit-exempt-hosts` is a comma-separated list of hostnames and IPs to be exempt from request rate 
  limiting; hostnames are resolved at the time the server is started. Defaults to an empty list.

### Message limits
By default, the number of messages a visitor can send is governed entirely by the [request limit](#request-limits). 
For instance, if the request limit allows for 15,000 requests per day, and all of those requests are POST/PUT requests
to publish messages, then that is the daily message limit.

To limit the number of daily messages per visitor, you can set `visitor-message-daily-limit`. This defines the number 
of messages a visitor can send in a day. This counter is reset every day at midnight (UTC).

### Attachment limits
Aside from the global file size and total attachment cache limits (see [above](#attachments)), there are two relevant 
per-visitor limits:

* `visitor-attachment-total-size-limit` is the total storage limit used for attachments per visitor. It defaults to 100M.
  The per-visitor storage is automatically decreased as attachments expire. External attachments (attached via `X-Attach`, 
  see [publishing docs](publish.md#attachments)) do not count here. 
* `visitor-attachment-daily-bandwidth-limit` is the total daily attachment download/upload bandwidth limit per visitor, 
  including PUT and GET requests. This is to protect your precious bandwidth from abuse, since egress costs money in
  most cloud providers. This defaults to 500M.

### E-mail limits
Similarly to the request limit, there is also an e-mail limit (only relevant if [e-mail notifications](#e-mail-notifications) 
are enabled):

* `visitor-email-limit-burst` is the initial bucket of emails each visitor has. This defaults to 16.
* `visitor-email-limit-replenish` is the rate at which the bucket is refilled (one email per x). Defaults to 1h.

### Firebase limits
If [Firebase is configured](#firebase-fcm), all messages are also published to a Firebase topic (unless `Firebase: no` 
is set). Firebase enforces [its own limits](https://firebase.google.com/docs/cloud-messaging/concept-options#topics_throttling)
on how many messages can be published. Unfortunately these limits are a little vague and can change depending on the time 
of day. In practice, I have only ever observed `429 Quota exceeded` responses from Firebase if **too many messages are published to 
the same topic**. 

In ntfy, if Firebase responds with a 429 after publishing to a topic, the visitor (= IP address) who published the message
is **banned from publishing to Firebase for 10 minutes** (not configurable). Because publishing to Firebase happens asynchronously,
there is no indication of the user that this has happened. Non-Firebase subscribers (WebSocket or HTTP stream) are not affected.
After the 10 minutes are up, messages forwarding to Firebase is resumed for this visitor.

If this ever happens, there will be a log message that looks something like this:
```
WARN Firebase quota exceeded (likely for topic), temporarily denying Firebase access to visitor
```

### IPv6 considerations
By default, rate limiting for IPv6 is done using the `/64` subnet of the visitor's IPv6 address. This means that all visitors
in the same `/64` subnet are treated as one visitor. This is done to prevent abuse, as IPv6 subnet assignments are typically
much larger than IPv4 subnets (and much cheaper), and it is common for ISPs to assign large subnets to their customers.

Other than that, rate limiting for IPv6 is done the same way as for IPv4, using the visitor's IP address or subnet to identify them.

There are two options to configure the number of bits used for rate limiting (for IPv4 and IPv6):

- `visitor-prefix-bits-ipv4` is number of bits of the IPv4 address to use for rate limiting (default: 32, full address)
- `visitor-prefix-bits-ipv6` is number of bits of the IPv6 address to use for rate limiting (default: 64, /64 subnet)

### Subscriber-based rate limiting
By default, ntfy puts almost all rate limits on the message publisher, e.g. number of messages, requests, and attachment
size are all based on the visitor who publishes a message. **Subscriber-based rate limiting is a way to use the rate limits
of a topic's subscriber, instead of the limits of the publisher.**

If subscriber-based rate limiting is enabled, **messages published on UnifiedPush topics** (topics starting with `up`, e.g. `up123456789012`) 
will be counted towards the "rate visitor" of the topic. A "rate visitor" is the first subscriber to the topic. 

Once enabled, a client subscribing to UnifiedPush topics via HTTP stream, or websockets, will be automatically registered as
a "rate visitor", i.e. the visitor whose rate limits will be used when publishing on this topic. Note that setting the rate visitor
requires **read-write permission** on the topic.

If this setting is enabled, publishing to UnifiedPush topics will lead to an `HTTP 507 Insufficient Storage`
response if no "rate visitor" has been previously registered. This is to avoid burning the publisher's 
`visitor-message-daily-limit`.

To enable subscriber-based rate limiting, set `visitor-subscriber-rate-limiting: true`.

!!! info
    Due to a [denial-of-service issue](https://github.com/binwiederhier/ntfy/issues/1048), support for the `Rate-Topics`
    header was removed entirely. This is unfortunate, but subscriber-based rate limiting will still work for `up*` topics.

## Tuning for scale
If you're running ntfy for your home server, you probably don't need to worry about scale at all. In its default config,
if it's not behind a proxy, the ntfy server can keep about **as many connections as the open file limit allows**.
This limit is typically called `nofile`. Other than that, RAM and CPU are obviously relevant. You may also want to check
out [this discussion on Reddit](https://www.reddit.com/r/golang/comments/r9u4ee/how_many_actively_connected_http_clients_can_a_go/).

Depending on *how you run it*, here are a few limits that are relevant:

### Message cache
By default, the [message cache](#message-cache) (defined by `cache-file`) uses the SQLite default settings, which means it
syncs to disk on every write. For personal servers, this is perfectly adequate. For larger installations, such as ntfy.sh,
the [write-ahead log (WAL)](https://sqlite.org/wal.html) should be enabled, and the sync mode should be adjusted. 
See [this article](https://phiresky.github.io/blog/2020/sqlite-performance-tuning/) for details.

In addition to that, for very high load servers (such as ntfy.sh), it may be beneficial to write messages to the cache
in batches, and asynchronously. This can be enabled with the `cache-batch-size` and `cache-batch-timeout`. If you start
seeing `database locked` messages in the logs, you should probably enable that.

Here's how ntfy.sh has been tuned in the `server.yml` file:

``` yaml
cache-batch-size: 25
cache-batch-timeout: "1s"
cache-startup-queries: |
    pragma journal_mode = WAL;
    pragma synchronous = normal;
    pragma temp_store = memory;
    pragma busy_timeout = 15000;
    vacuum;
```

### For systemd services
If you're running ntfy in a systemd service (e.g. for .deb/.rpm packages), the main limiting factor is the
`LimitNOFILE` setting in the systemd unit. The default open files limit for `ntfy.service` is 10,000. You can override it
by creating a `/etc/systemd/system/ntfy.service.d/override.conf` file. As far as I can tell, `/etc/security/limits.conf`
is not relevant.

=== "/etc/systemd/system/ntfy.service.d/override.conf"
    ```
    # Allow 20,000 ntfy connections (and give room for other file handles)
    [Service]
    LimitNOFILE=20500
    ```

### Outside of systemd
If you're running outside systemd, you may want to adjust your `/etc/security/limits.conf` file to
increase the `nofile` setting. Here's an example that increases the limit to 5,000. You can find out the current setting
by running `ulimit -n`, or manually override it temporarily by running `ulimit -n 50000`.

=== "/etc/security/limits.conf"
    ```
    # Increase open files limit globally
    * hard nofile 20500
    ```

### Proxy limits (nginx, Apache2)
If you are running [behind a proxy](#behind-a-proxy-tls-etc) (e.g. nginx, Apache), the open files limit of the proxy is also
relevant. So if your proxy runs inside of systemd, increase the limits in systemd for the proxy. Typically, the proxy
open files limit has to be **double the number of how many connections you'd like to support**, because the proxy has
to maintain the client connection and the connection to ntfy.

=== "/etc/nginx/nginx.conf"
    ```
    events {
      # Allow 40,000 proxy connections (2x of the desired ntfy connection count;
      # and give room for other file handles)
      worker_connections 40500;
    }
    ```

=== "/etc/systemd/system/nginx.service.d/override.conf"
    ```
    # Allow 40,000 proxy connections (2x of the desired ntfy connection count;
    # and give room for other file handles)
    [Service]
    LimitNOFILE=40500
    ```

### Banning bad actors (fail2ban)
If you put stuff on the Internet, bad actors will try to break them or break in. [fail2ban](https://www.fail2ban.org/)
and nginx's [ngx_http_limit_req_module module](http://nginx.org/en/docs/http/ngx_http_limit_req_module.html) can be used
to ban client IPs if they misbehave. This is on top of the [rate limiting](#rate-limiting) inside the ntfy server.

Here's an example for how ntfy.sh is configured, following the instructions from two tutorials ([here](https://easyengine.io/tutorials/nginx/fail2ban/) 
and [here](https://easyengine.io/tutorials/nginx/block-wp-login-php-bruteforce-attack/)):

=== "/etc/nginx/nginx.conf"
    ```
    # Rate limit all IP addresses
    http {
	  limit_req_zone $binary_remote_addr zone=one:10m rate=45r/m;
    }

    # Alternatively, whitelist certain IP addresses
    http {
      geo $limited {
        default 1;
        116.203.112.46/32 0;
        132.226.42.65/32 0;
        ...
      }
      map $limited $limitkey {
        1 $binary_remote_addr;
        0 "";
      }
      limit_req_zone $limitkey zone=one:10m rate=45r/m;
    }
    ```

=== "/etc/nginx/sites-enabled/ntfy.sh"
    ```
    # For each server/location block
    server {
      location / {
        limit_req zone=one burst=1000 nodelay;
      }
    }    
    ```

=== "/etc/fail2ban/filter.d/nginx-req-limit.conf"
    ```
    [Definition]
    failregex = limiting requests, excess:.* by zone.*client: <HOST>
    ignoreregex =
    ```

=== "/etc/fail2ban/jail.local"
    ```
    [nginx-req-limit]
    enabled = true
    filter = nginx-req-limit
    action = iptables-multiport[name=ReqLimit, port="http,https", protocol=tcp]
    logpath = /var/log/nginx/error.log
    findtime = 600
    bantime = 14400
    maxretry = 10
    ```

Note that if you run nginx in a container, append `, chain=DOCKER-USER` to the jail.local action. By default, the jail action chain
is `INPUT`, but `FORWARD` is used when using docker networks. `DOCKER-USER`, available when using docker, is part of the `FORWARD`
chain.

The official ntfy.sh server uses fail2ban to ban IPs. Check out ntfy.sh's [Ansible fail2ban role](https://github.com/binwiederhier/ntfy-ansible/tree/main/roles/fail2ban) for details. Ban actors are banned for 1 hour initially, and up to
4 hours at a time for repeated offenses. IPv4 addresses are banned individually, while IPv6 addresses are banned by their `/56` prefix.

## IPv6 support
ntfy fully supports IPv6, though there are a few things to keep in mind.

- **Listening on an IPv6 address**: By default, ntfy listens on `:80` (IPv4-only). If you want to listen on an IPv6 address, you need to
  explicitly set the `listen-http` and/or `listen-https` options in your `server.yml` file to an IPv6 address, e.g. `[::]:80`. To listen on
  IPv4 and IPv6, you must run ntfy behind a reverse proxy, e.g. `listen :80; listen [::]:80;` in nginx.
- **Rate limiting:** By default, ntfy uses the `/64` subnet of the visitor's IPv6 address for rate limiting. This means that all visitors in the same `/64`
  subnet are treated as one visitor. If you want to change this, you can set the `visitor-prefix-bits-ipv6` option in your `server.yml` file to a different
  value (e.g. `48` for `/48` subnets). See [IPv6 considerations](#ipv6-considerations) and [IP-based rate limiting](#ip-based-rate-limiting) for more details.
- **Banning IPs with fail2ban:** By default, if you're using the `iptables-multiport` action, fail2ban bans individual IPv4 and IPv6 addresses via `iptables` and `ip6tables`. While this behavior is fine for IPv4, it is not for IPv6, because every host can technically have up to 2^64 addresses. Please ensure that your `actionban` and `actionunban` commands
  support IPv6 and also ban the entire prefix (e.g. `/48`). See [Banning bad actors](#banning-bad-actors-fail2ban) for details.

!!! info
    The official ntfy.sh server supports IPv6. Check out ntfy.sh's [Ansible repository](https://github.com/binwiederhier/ntfy-ansible) for examples of how to
    configure [ntfy](https://github.com/binwiederhier/ntfy-ansible/tree/main/roles/ntfy), [nginx](https://github.com/binwiederhier/ntfy-ansible/tree/main/roles/nginx) and [fail2ban](https://github.com/binwiederhier/ntfy-ansible/tree/main/roles/fail2ban).

## Health checks
A preliminary health check API endpoint is exposed at `/v1/health`. The endpoint returns a `json` response in the format shown below.
If a non-200 HTTP status code is returned or if the returned `healthy` field is `false` the ntfy service should be considered as unhealthy.

```json
{"healthy":true}
```

See [Installation for Docker](install.md#docker) for an example of how this could be used in a `docker-compose` environment.

## Monitoring
If configured, ntfy can expose a `/metrics` endpoint for [Prometheus](https://prometheus.io/), which can then be used to
create dashboards and alerts (e.g. via [Grafana](https://grafana.com/)).

To configure the metrics endpoint, either set `enable-metrics` and/or set the `listen-metrics-http` option to a dedicated
listen address. Metrics may be considered sensitive information, so before you enable them, be sure you know what you are
doing, and/or secure access to the endpoint in your reverse proxy.

- `enable-metrics` enables the /metrics endpoint for the default ntfy server (i.e. HTTP, HTTPS and/or Unix socket)
- `metrics-listen-http` exposes the metrics endpoint via a dedicated `[IP]:port`. If set, this option implicitly
  enables metrics as well, e.g. "10.0.1.1:9090" or ":9090"

=== "server.yml (Using default port)"
    ```yaml
    enable-metrics: true
    ```

=== "server.yml (Using dedicated IP/port)"
    ```yaml
    metrics-listen-http: "10.0.1.1:9090"
    ```

In Prometheus, an example scrape config would look like this:

=== "prometheus.yml"
    ```yaml
    scrape_configs:
      - job_name: "ntfy"
        static_configs:
          - targets: ["10.0.1.1:9090"]
    ```

Here's an example Grafana dashboard built from the metrics (see [Grafana JSON on GitHub](https://raw.githubusercontent.com/binwiederhier/ntfy/main/examples/grafana-dashboard/ntfy-grafana.json)):

<figure markdown style="padding-left: 50px; padding-right: 50px">
  <a href="../../static/img/grafana-dashboard.png" target="_blank"><img src="../../static/img/grafana-dashboard.png"/></a>
  <figcaption>ntfy Grafana dashboard</figcaption>
</figure>

## Profiling
ntfy can expose Go's [net/http/pprof](https://pkg.go.dev/net/http/pprof) endpoints to support profiling of the ntfy server. 
If enabled, ntfy will listen on a dedicated listen IP/port, which can be accessed via the web browser on `http://<ip>:<port>/debug/pprof/`.
This can be helpful to expose bottlenecks, and visualize call flows. To enable, simply set the `profile-listen-http` config option.

## Logging & debugging
By default, ntfy logs to the console (stderr), with an `info` log level, and in a human-readable text format.

ntfy supports five different log levels, can also write to a file, log as JSON, and even supports granular
log level overrides for easier debugging. Some options (`log-level` and `log-level-overrides`) can be hot reloaded
by calling `kill -HUP $pid` or `systemctl reload ntfy`.

The following config options define the logging behavior:

* `log-format` defines the output format, can be `text` (default) or `json`
* `log-file` is a filename to write logs to. If this is not set, ntfy logs to stderr.
* `log-level` defines the default log level, can be one of `trace`, `debug`, `info` (default), `warn` or `error`.
  Be aware that `debug` (and particularly `trace`) can be **very verbose**. Only turn them on briefly for debugging purposes.
* `log-level-overrides` lets you override the log level if certain fields match. This is incredibly powerful
  for debugging certain parts of the system (e.g. only the account management, or only a certain visitor).
  This is an array of strings in the format:
    - `field=value -> level` to match a value exactly, e.g. `tag=manager -> trace`
    - `field -> level` to match any value, e.g. `time_taken_ms -> debug`

**Logging config (good for production use):**
``` yaml
log-level: info
log-format: json
log-file: /var/log/ntfy.log
```

**Temporary debugging:**   
If something's not working right, you can debug/trace through what the ntfy server is doing by setting the `log-level`
to `debug` or `trace`. The `debug` setting will output information about each published message, but not the message
contents. The `trace` setting will also print the message contents.

Alternatively, you can set `log-level-overrides` for only certain fields, such as a visitor's IP address (`visitor_ip`), 
a username (`user_name`), or a tag (`tag`). There are dozens of fields you can use to override log levels. To learn what 
they are, either turn the log-level to `trace` and observe, or reference the [source code](https://github.com/binwiederhier/ntfy).

Here's an example that will output only `info` log events, except when they match either of the defined overrides:
``` yaml
log-level: info
log-level-overrides:
  - "tag=manager -> trace"
  - "visitor_ip=1.2.3.4 -> debug"
  - "time_taken_ms -> debug"
```

!!! warning
    The `debug` and `trace` log levels are very verbose, and using `log-level-overrides` has a 
    performance penalty. Only use it for temporary debugging.

You can also hot-reload the `log-level` and `log-level-overrides` by sending the `SIGHUP` signal to the process after 
editing the `server.yml` file. You can do so by calling `systemctl reload ntfy` (if ntfy is running inside systemd), 
or by calling `kill -HUP $(pidof ntfy)`. If successful, you'll see something like this:

```
$ ntfy serve
2022/06/02 10:29:28 INFO Listening on :2586[http] :1025[smtp], log level is INFO
2022/06/02 10:29:34 INFO Partially hot reloading configuration ...
2022/06/02 10:29:34 INFO Log level is TRACE
```

## Config options
Each config option can be set in the config file `/etc/ntfy/server.yml` (e.g. `listen-http: :80`) or as a
CLI option (e.g. `--listen-http :80`. Here's a list of all available options. Alternatively, you can set an environment
variable before running the `ntfy` command (e.g. `export NTFY_LISTEN_HTTP=:80`).

!!! info
    All config options can also be defined in the `server.yml` file using underscores instead of dashes, e.g. 
    `cache_duration` and `cache-duration` are both supported. This is to support stricter YAML parsers that do 
    not support dashes.

| Config option                              | Env variable                                    | Format                                              | Default           | Description                                                                                                                                                                                                                     |
|--------------------------------------------|-------------------------------------------------|-----------------------------------------------------|-------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `base-url`                                 | `NTFY_BASE_URL`                                 | *URL*                                               | -                 | Public facing base URL of the service (e.g. `https://ntfy.sh`)                                                                                                                                                                  |
| `listen-http`                              | `NTFY_LISTEN_HTTP`                              | `[host]:port`                                       | `:80`             | Listen address for the HTTP web server                                                                                                                                                                                          |
| `listen-https`                             | `NTFY_LISTEN_HTTPS`                             | `[host]:port`                                       | -                 | Listen address for the HTTPS web server. If set, you also need to set `key-file` and `cert-file`.                                                                                                                               |
| `listen-unix`                              | `NTFY_LISTEN_UNIX`                              | *filename*                                          | -                 | Path to a Unix socket to listen on                                                                                                                                                                                              |
| `listen-unix-mode`                         | `NTFY_LISTEN_UNIX_MODE`                         | *file mode*                                         | *system default*  | File mode of the Unix socket, e.g. 0700 or 0777                                                                                                                                                                                 |
| `key-file`                                 | `NTFY_KEY_FILE`                                 | *filename*                                          | -                 | HTTPS/TLS private key file, only used if `listen-https` is set.                                                                                                                                                                 |
| `cert-file`                                | `NTFY_CERT_FILE`                                | *filename*                                          | -                 | HTTPS/TLS certificate file, only used if `listen-https` is set.                                                                                                                                                                 |
| `firebase-key-file`                        | `NTFY_FIREBASE_KEY_FILE`                        | *filename*                                          | -                 | If set, also publish messages to a Firebase Cloud Messaging (FCM) topic for your app. This is optional and only required to save battery when using the Android app. See [Firebase (FCM)](#firebase-fcm).                       |
| `cache-file`                               | `NTFY_CACHE_FILE`                               | *filename*                                          | -                 | If set, messages are cached in a local SQLite database instead of only in-memory. This allows for service restarts without losing messages in support of the since= parameter. See [message cache](#message-cache).             |
| `cache-duration`                           | `NTFY_CACHE_DURATION`                           | *duration*                                          | 12h               | Duration for which messages will be buffered before they are deleted. This is required to support the `since=...` and `poll=1` parameter. Set this to `0` to disable the cache entirely.                                        |
| `cache-startup-queries`                    | `NTFY_CACHE_STARTUP_QUERIES`                    | *string (SQL queries)*                              | -                 | SQL queries to run during database startup; this is useful for tuning and [enabling WAL mode](#message-cache)                                                                                                                   |
| `cache-batch-size`                         | `NTFY_CACHE_BATCH_SIZE`                         | *int*                                               | 0                 | Max size of messages to batch together when writing to message cache (if zero, writes are synchronous)                                                                                                                          |
| `cache-batch-timeout`                      | `NTFY_CACHE_BATCH_TIMEOUT`                      | *duration*                                          | 0s                | Timeout for batched async writes to the message cache (if zero, writes are synchronous)                                                                                                                                         |
| `auth-file`                                | `NTFY_AUTH_FILE`                                | *filename*                                          | -                 | Auth database file used for access control. If set, enables authentication and access control. See [access control](#access-control).                                                                                           |
| `auth-default-access`                      | `NTFY_AUTH_DEFAULT_ACCESS`                      | `read-write`, `read-only`, `write-only`, `deny-all` | `read-write`      | Default permissions if no matching entries in the auth database are found. Default is `read-write`.                                                                                                                             |
| `behind-proxy`                             | `NTFY_BEHIND_PROXY`                             | *bool*                                              | false             | If set, use forwarded header (e.g. X-Forwarded-For, X-Client-IP) to determine visitor IP address (for rate limiting)                                                                                                            |
| `proxy-forwarded-header`                   | `NTFY_PROXY_FORWARDED_HEADER`                   | *string*                                            | `X-Forwarded-For` | Use specified header to determine visitor IP address (for rate limiting)                                                                                                                                                        |
| `proxy-trusted-hosts`                      | `NTFY_PROXY_TRUSTED_HOSTS`                      | *comma-separated host/IP/CIDR list*                 | -                 | Comma-separated list of trusted IP addresses, hosts, or CIDRs to remove from forwarded header                                                                                                                                   |
| `attachment-cache-dir`                     | `NTFY_ATTACHMENT_CACHE_DIR`                     | *directory*                                         | -                 | Cache directory for attached files. To enable attachments, this has to be set.                                                                                                                                                  |
| `attachment-total-size-limit`              | `NTFY_ATTACHMENT_TOTAL_SIZE_LIMIT`              | *size*                                              | 5G                | Limit of the on-disk attachment cache directory. If the limits is exceeded, new attachments will be rejected.                                                                                                                   |
| `attachment-file-size-limit`               | `NTFY_ATTACHMENT_FILE_SIZE_LIMIT`               | *size*                                              | 15M               | Per-file attachment size limit (e.g. 300k, 2M, 100M). Larger attachment will be rejected.                                                                                                                                       |
| `attachment-expiry-duration`               | `NTFY_ATTACHMENT_EXPIRY_DURATION`               | *duration*                                          | 3h                | Duration after which uploaded attachments will be deleted (e.g. 3h, 20h). Strongly affects `visitor-attachment-total-size-limit`.                                                                                               |
| `smtp-sender-addr`                         | `NTFY_SMTP_SENDER_ADDR`                         | `host:port`                                         | -                 | SMTP server address to allow email sending                                                                                                                                                                                      |
| `smtp-sender-user`                         | `NTFY_SMTP_SENDER_USER`                         | *string*                                            | -                 | SMTP user; only used if e-mail sending is enabled                                                                                                                                                                               |
| `smtp-sender-pass`                         | `NTFY_SMTP_SENDER_PASS`                         | *string*                                            | -                 | SMTP password; only used if e-mail sending is enabled                                                                                                                                                                           |
| `smtp-sender-from`                         | `NTFY_SMTP_SENDER_FROM`                         | *e-mail address*                                    | -                 | SMTP sender e-mail address; only used if e-mail sending is enabled                                                                                                                                                              |
| `smtp-server-listen`                       | `NTFY_SMTP_SERVER_LISTEN`                       | `[ip]:port`                                         | -                 | Defines the IP address and port the SMTP server will listen on, e.g. `:25` or `1.2.3.4:25`                                                                                                                                      |
| `smtp-server-domain`                       | `NTFY_SMTP_SERVER_DOMAIN`                       | *domain name*                                       | -                 | SMTP server e-mail domain, e.g. `ntfy.sh`                                                                                                                                                                                       |
| `smtp-server-addr-prefix`                  | `NTFY_SMTP_SERVER_ADDR_PREFIX`                  | *string*                                            | -                 | Optional prefix for the e-mail addresses to prevent spam, e.g. `ntfy-`                                                                                                                                                          |
| `twilio-account`                           | `NTFY_TWILIO_ACCOUNT`                           | *string*                                            | -                 | Twilio account SID, e.g. AC12345beefbeef67890beefbeef122586                                                                                                                                                                     |
| `twilio-auth-token`                        | `NTFY_TWILIO_AUTH_TOKEN`                        | *string*                                            | -                 | Twilio auth token, e.g. affebeef258625862586258625862586                                                                                                                                                                        |
| `twilio-phone-number`                      | `NTFY_TWILIO_PHONE_NUMBER`                      | *string*                                            | -                 | Twilio outgoing phone number, e.g. +18775132586                                                                                                                                                                                 |
| `twilio-verify-service`                    | `NTFY_TWILIO_VERIFY_SERVICE`                    | *string*                                            | -                 | Twilio Verify service SID, e.g. VA12345beefbeef67890beefbeef122586                                                                                                                                                              |
| `keepalive-interval`                       | `NTFY_KEEPALIVE_INTERVAL`                       | *duration*                                          | 45s               | Interval in which keepalive messages are sent to the client. This is to prevent intermediaries closing the connection for inactivity. Note that the Android app has a hardcoded timeout at 77s, so it should be less than that. |
| `manager-interval`                         | `NTFY_MANAGER_INTERVAL`                         | *duration*                                          | 1m                | Interval in which the manager prunes old messages, deletes topics and prints the stats.                                                                                                                                         |
| `message-size-limit`                       | `NTFY_MESSAGE_SIZE_LIMIT`                       | *size*                                              | 4K                | The size limit for the message body. Please note that this is largely untested, and that FCM/APNS have limits around 4KB. If you increase this size limit, FCM and APNS will NOT work for large messages.                       |
| `message-delay-limit`                      | `NTFY_MESSAGE_DELAY_LIMIT`                      | *duration*                                          | 3d                | Amount of time a message can be [scheduled](publish.md#scheduled-delivery) into the future when using the `Delay` header                                                                                                        |
| `global-topic-limit`                       | `NTFY_GLOBAL_TOPIC_LIMIT`                       | *number*                                            | 15,000            | Rate limiting: Total number of topics before the server rejects new topics.                                                                                                                                                     |
| `upstream-base-url`                        | `NTFY_UPSTREAM_BASE_URL`                        | *URL*                                               | `https://ntfy.sh` | Forward poll request to an upstream server, this is needed for iOS push notifications for self-hosted servers                                                                                                                   |
| `upstream-access-token`                    | `NTFY_UPSTREAM_ACCESS_TOKEN`                    | *string*                                            | `tk_zyYLYj...`    | Access token to use for the upstream server; needed only if upstream rate limits are exceeded or upstream server requires auth                                                                                                  |
| `visitor-attachment-total-size-limit`      | `NTFY_VISITOR_ATTACHMENT_TOTAL_SIZE_LIMIT`      | *size*                                              | 100M              | Rate limiting: Total storage limit used for attachments per visitor, for all attachments combined. Storage is freed after attachments expire. See `attachment-expiry-duration`.                                                 |
| `visitor-attachment-daily-bandwidth-limit` | `NTFY_VISITOR_ATTACHMENT_DAILY_BANDWIDTH_LIMIT` | *size*                                              | 500M              | Rate limiting: Total daily attachment download/upload traffic limit per visitor. This is to protect your bandwidth costs from exploding.                                                                                        |
| `visitor-email-limit-burst`                | `NTFY_VISITOR_EMAIL_LIMIT_BURST`                | *number*                                            | 16                | Rate limiting:Initial limit of e-mails per visitor                                                                                                                                                                              |
| `visitor-email-limit-replenish`            | `NTFY_VISITOR_EMAIL_LIMIT_REPLENISH`            | *duration*                                          | 1h                | Rate limiting: Strongly related to `visitor-email-limit-burst`: The rate at which the bucket is refilled                                                                                                                        |
| `visitor-message-daily-limit`              | `NTFY_VISITOR_MESSAGE_DAILY_LIMIT`              | *number*                                            | -                 | Rate limiting: Allowed number of messages per day per visitor, reset every day at midnight (UTC). By default, this value is unset.                                                                                              |
| `visitor-request-limit-burst`              | `NTFY_VISITOR_REQUEST_LIMIT_BURST`              | *number*                                            | 60                | Rate limiting: Allowed GET/PUT/POST requests per second, per visitor. This setting is the initial bucket of requests each visitor has                                                                                           |
| `visitor-request-limit-replenish`          | `NTFY_VISITOR_REQUEST_LIMIT_REPLENISH`          | *duration*                                          | 5s                | Rate limiting: Strongly related to `visitor-request-limit-burst`: The rate at which the bucket is refilled                                                                                                                      |
| `visitor-request-limit-exempt-hosts`       | `NTFY_VISITOR_REQUEST_LIMIT_EXEMPT_HOSTS`       | *comma-separated host/IP/CIDR list*                 | -                 | Rate limiting: List of hostnames and IPs to be exempt from request rate limiting                                                                                                                                                |
| `visitor-subscription-limit`               | `NTFY_VISITOR_SUBSCRIPTION_LIMIT`               | *number*                                            | 30                | Rate limiting: Number of subscriptions per visitor (IP address)                                                                                                                                                                 |
| `visitor-subscriber-rate-limiting`         | `NTFY_VISITOR_SUBSCRIBER_RATE_LIMITING`         | *bool*                                              | `false`           | Rate limiting: Enables subscriber-based rate limiting                                                                                                                                                                           |
| `visitor-prefix-bits-ipv4`                 | `NTFY_VISITOR_PREFIX_BITS_IPV4`                 | *number*                                            | 32                | Rate limiting: Number of bits to use for IPv4 visitor prefix, e.g. 24 for /24                                                                                                                                                   |
| `visitor-prefix-bits-ipv6`                 | `NTFY_VISITOR_PREFIX_BITS_IPV6`                 | *number*                                            | 64                | Rate limiting: Number of bits to use for IPv6 visitor prefix, e.g. 48 for /48                                                                                                                                                   |
| `web-root`                                 | `NTFY_WEB_ROOT`                                 | *path*, e.g. `/` or `/app`, or `disable`            | `/`               | Sets root of the web app (e.g. /, or /app), or disables it entirely (disable)                                                                                                                                                   |
| `enable-signup`                            | `NTFY_ENABLE_SIGNUP`                            | *boolean* (`true` or `false`)                       | `false`           | Allows users to sign up via the web app, or API                                                                                                                                                                                 |
| `enable-login`                             | `NTFY_ENABLE_LOGIN`                             | *boolean* (`true` or `false`)                       | `false`           | Allows users to log in via the web app, or API                                                                                                                                                                                  |
| `enable-reservations`                      | `NTFY_ENABLE_RESERVATIONS`                      | *boolean* (`true` or `false`)                       | `false`           | Allows users to reserve topics (if their tier allows it)                                                                                                                                                                        |
| `stripe-secret-key`                        | `NTFY_STRIPE_SECRET_KEY`                        | *string*                                            | -                 | Payments: Key used for the Stripe API communication, this enables payments                                                                                                                                                      |
| `stripe-webhook-key`                       | `NTFY_STRIPE_WEBHOOK_KEY`                       | *string*                                            | -                 | Payments: Key required to validate the authenticity of incoming webhooks from Stripe                                                                                                                                            |
| `billing-contact`                          | `NTFY_BILLING_CONTACT`                          | *email address* or *website*                        | -                 | Payments: Email or website displayed in Upgrade dialog as a billing contact                                                                                                                                                     |
| `web-push-public-key`                      | `NTFY_WEB_PUSH_PUBLIC_KEY`                      | *string*                                            | -                 | Web Push: Public Key. Run `ntfy webpush keys` to generate                                                                                                                                                                       |
| `web-push-private-key`                     | `NTFY_WEB_PUSH_PRIVATE_KEY`                     | *string*                                            | -                 | Web Push: Private Key. Run `ntfy webpush keys` to generate                                                                                                                                                                      |
| `web-push-file`                            | `NTFY_WEB_PUSH_FILE`                            | *string*                                            | -                 | Web Push: Database file that stores subscriptions                                                                                                                                                                               |
| `web-push-email-address`                   | `NTFY_WEB_PUSH_EMAIL_ADDRESS`                   | *string*                                            | -                 | Web Push: Sender email address                                                                                                                                                                                                  |
| `web-push-startup-queries`                 | `NTFY_WEB_PUSH_STARTUP_QUERIES`                 | *string*                                            | -                 | Web Push: SQL queries to run against subscription database at startup                                                                                                                                                           |
| `web-push-expiry-duration`                 | `NTFY_WEB_PUSH_EXPIRY_DURATION`                 | *duration*                                          | 60d               | Web Push: Duration after which a subscription is considered stale and will be deleted. This is to prevent stale subscriptions.                                                                                                  |
| `web-push-expiry-warning-duration`         | `NTFY_WEB_PUSH_EXPIRY_WARNING_DURATION`         | *duration*                                          | 55d               | Web Push: Duration after which a warning is sent to subscribers that their subscription will expire soon. This is to prevent stale subscriptions.                                                                               |
| `log-format`                               | `NTFY_LOG_FORMAT`                               | *string*                                            | `text`            | Defines the output format, can be text or json                                                                                                                                                                                  |
| `log-file`                                 | `NTFY_LOG_FILE`                                 | *string*                                            | -                 | Defines the filename to write logs to. If this is not set, ntfy logs to stderr                                                                                                                                                  |
| `log-level`                                | `NTFY_LOG_LEVEL`                                | *string*                                            | `info`            | Defines the default log level, can be one of trace, debug, info, warn or error                                                                                                                                                  |

The format for a *duration* is: `<number>(smhd)`, e.g. 30s, 20m, 1h or 3d.   
The format for a *size* is: `<number>(GMK)`, e.g. 1G, 200M or 4000k.

## Command line options
```
NAME:
   ntfy serve - Run the ntfy server

USAGE:
   ntfy serve [OPTIONS..]

CATEGORY:
   Server commands

DESCRIPTION:
   Run the ntfy server and listen for incoming requests

   The command will load the configuration from /etc/ntfy/server.yml. Config options can 
   be overridden using the command line options.

   Examples:
     ntfy serve                      # Starts server in the foreground (on port 80)
     ntfy serve --listen-http :8080  # Starts server with alternate port

OPTIONS:
   --debug, -d                                                                                                            enable debug logging (default: false) [$NTFY_DEBUG]
   --trace                                                                                                                enable tracing (very verbose, be careful) (default: false) [$NTFY_TRACE]
   --no-log-dates, --no_log_dates                                                                                         disable the date/time prefix (default: false) [$NTFY_NO_LOG_DATES]
   --log-level value, --log_level value                                                                                   set log level (default: "INFO") [$NTFY_LOG_LEVEL]
   --log-level-overrides value, --log_level_overrides value [ --log-level-overrides value, --log_level_overrides value ]  set log level overrides [$NTFY_LOG_LEVEL_OVERRIDES]
   --log-format value, --log_format value                                                                                 set log format (default: "text") [$NTFY_LOG_FORMAT]
   --log-file value, --log_file value                                                                                     set log file, default is STDOUT [$NTFY_LOG_FILE]
   --config value, -c value                                                                                               config file (default: "/etc/ntfy/server.yml") [$NTFY_CONFIG_FILE]
   --base-url value, --base_url value, -B value                                                                           externally visible base URL for this host (e.g. https://ntfy.sh) [$NTFY_BASE_URL]
   --listen-http value, --listen_http value, -l value                                                                     ip:port used as HTTP listen address (default: ":80") [$NTFY_LISTEN_HTTP]
   --listen-https value, --listen_https value, -L value                                                                   ip:port used as HTTPS listen address [$NTFY_LISTEN_HTTPS]
   --listen-unix value, --listen_unix value, -U value                                                                     listen on unix socket path [$NTFY_LISTEN_UNIX]
   --listen-unix-mode value, --listen_unix_mode value                                                                     file permissions of unix socket, e.g. 0700 (default: system default) [$NTFY_LISTEN_UNIX_MODE]
   --key-file value, --key_file value, -K value                                                                           private key file, if listen-https is set [$NTFY_KEY_FILE]
   --cert-file value, --cert_file value, -E value                                                                         certificate file, if listen-https is set [$NTFY_CERT_FILE]
   --firebase-key-file value, --firebase_key_file value, -F value                                                         Firebase credentials file; if set additionally publish to FCM topic [$NTFY_FIREBASE_KEY_FILE]
   --cache-file value, --cache_file value, -C value                                                                       cache file used for message caching [$NTFY_CACHE_FILE]
   --cache-duration since, --cache_duration since, -b since                                                               buffer messages for this time to allow since requests (default: "12h") [$NTFY_CACHE_DURATION]
   --cache-batch-size value, --cache_batch_size value                                                                     max size of messages to batch together when writing to message cache (if zero, writes are synchronous) (default: 0) [$NTFY_BATCH_SIZE]
   --cache-batch-timeout value, --cache_batch_timeout value                                                               timeout for batched async writes to the message cache (if zero, writes are synchronous) (default: "0s") [$NTFY_CACHE_BATCH_TIMEOUT]
   --cache-startup-queries value, --cache_startup_queries value                                                           queries run when the cache database is initialized [$NTFY_CACHE_STARTUP_QUERIES]
   --auth-file value, --auth_file value, -H value                                                                         auth database file used for access control [$NTFY_AUTH_FILE]
   --auth-startup-queries value, --auth_startup_queries value                                                             queries run when the auth database is initialized [$NTFY_AUTH_STARTUP_QUERIES]
   --auth-default-access value, --auth_default_access value, -p value                                                     default permissions if no matching entries in the auth database are found (default: "read-write") [$NTFY_AUTH_DEFAULT_ACCESS]
   --attachment-cache-dir value, --attachment_cache_dir value                                                             cache directory for attached files [$NTFY_ATTACHMENT_CACHE_DIR]
   --attachment-total-size-limit value, --attachment_total_size_limit value, -A value                                     limit of the on-disk attachment cache (default: "5G") [$NTFY_ATTACHMENT_TOTAL_SIZE_LIMIT]
   --attachment-file-size-limit value, --attachment_file_size_limit value, -Y value                                       per-file attachment size limit (e.g. 300k, 2M, 100M) (default: "15M") [$NTFY_ATTACHMENT_FILE_SIZE_LIMIT]
   --attachment-expiry-duration value, --attachment_expiry_duration value, -X value                                       duration after which uploaded attachments will be deleted (e.g. 3h, 20h) (default: "3h") [$NTFY_ATTACHMENT_EXPIRY_DURATION]
   --keepalive-interval value, --keepalive_interval value, -k value                                                       interval of keepalive messages (default: "45s") [$NTFY_KEEPALIVE_INTERVAL]
   --manager-interval value, --manager_interval value, -m value                                                           interval of for message pruning and stats printing (default: "1m") [$NTFY_MANAGER_INTERVAL]
   --disallowed-topics value, --disallowed_topics value [ --disallowed-topics value, --disallowed_topics value ]          topics that are not allowed to be used [$NTFY_DISALLOWED_TOPICS]
   --web-root value, --web_root value                                                                                     sets root of the web app (e.g. /, or /app), or disables it (disable) (default: "/") [$NTFY_WEB_ROOT]
   --enable-signup, --enable_signup                                                                                       allows users to sign up via the web app, or API (default: false) [$NTFY_ENABLE_SIGNUP]
   --enable-login, --enable_login                                                                                         allows users to log in via the web app, or API (default: false) [$NTFY_ENABLE_LOGIN]
   --enable-reservations, --enable_reservations                                                                           allows users to reserve topics (if their tier allows it) (default: false) [$NTFY_ENABLE_RESERVATIONS]
   --upstream-base-url value, --upstream_base_url value                                                                   forward poll request to an upstream server, this is needed for iOS push notifications for self-hosted servers [$NTFY_UPSTREAM_BASE_URL]
   --upstream-access-token value, --upstream_access_token value                                                           access token to use for the upstream server; needed only if upstream rate limits are exceeded or upstream server requires auth [$NTFY_UPSTREAM_ACCESS_TOKEN]
   --smtp-sender-addr value, --smtp_sender_addr value                                                                     SMTP server address (host:port) for outgoing emails [$NTFY_SMTP_SENDER_ADDR]
   --smtp-sender-user value, --smtp_sender_user value                                                                     SMTP user (if e-mail sending is enabled) [$NTFY_SMTP_SENDER_USER]
   --smtp-sender-pass value, --smtp_sender_pass value                                                                     SMTP password (if e-mail sending is enabled) [$NTFY_SMTP_SENDER_PASS]
   --smtp-sender-from value, --smtp_sender_from value                                                                     SMTP sender address (if e-mail sending is enabled) [$NTFY_SMTP_SENDER_FROM]
   --smtp-server-listen value, --smtp_server_listen value                                                                 SMTP server address (ip:port) for incoming emails, e.g. :25 [$NTFY_SMTP_SERVER_LISTEN]
   --smtp-server-domain value, --smtp_server_domain value                                                                 SMTP domain for incoming e-mail, e.g. ntfy.sh [$NTFY_SMTP_SERVER_DOMAIN]
   --smtp-server-addr-prefix value, --smtp_server_addr_prefix value                                                       SMTP email address prefix for topics to prevent spam (e.g. 'ntfy-') [$NTFY_SMTP_SERVER_ADDR_PREFIX]
   --twilio-account value, --twilio_account value                                                                         Twilio account SID, used for phone calls, e.g. AC123... [$NTFY_TWILIO_ACCOUNT]
   --twilio-auth-token value, --twilio_auth_token value                                                                   Twilio auth token [$NTFY_TWILIO_AUTH_TOKEN]
   --twilio-phone-number value, --twilio_phone_number value                                                               Twilio number to use for outgoing calls [$NTFY_TWILIO_PHONE_NUMBER]
   --twilio-verify-service value, --twilio_verify_service value                                                           Twilio Verify service ID, used for phone number verification [$NTFY_TWILIO_VERIFY_SERVICE]
   --message-size-limit value, --message_size_limit value                                                                 size limit for the message (see docs for limitations) (default: "4K") [$NTFY_MESSAGE_SIZE_LIMIT]
   --message-delay-limit value, --message_delay_limit value                                                               max duration a message can be scheduled into the future (default: "3d") [$NTFY_MESSAGE_DELAY_LIMIT]
   --global-topic-limit value, --global_topic_limit value, -T value                                                       total number of topics allowed (default: 15000) [$NTFY_GLOBAL_TOPIC_LIMIT]
   --visitor-subscription-limit value, --visitor_subscription_limit value                                                 number of subscriptions per visitor (default: 30) [$NTFY_VISITOR_SUBSCRIPTION_LIMIT]
   --visitor-subscriber-rate-limiting, --visitor_subscriber_rate_limiting                                                 enables subscriber-based rate limiting (default: false) [$NTFY_VISITOR_SUBSCRIBER_RATE_LIMITING]
   --visitor-attachment-total-size-limit value, --visitor_attachment_total_size_limit value                               total storage limit used for attachments per visitor (default: "100M") [$NTFY_VISITOR_ATTACHMENT_TOTAL_SIZE_LIMIT]
   --visitor-attachment-daily-bandwidth-limit value, --visitor_attachment_daily_bandwidth_limit value                     total daily attachment download/upload bandwidth limit per visitor (default: "500M") [$NTFY_VISITOR_ATTACHMENT_DAILY_BANDWIDTH_LIMIT]
   --visitor-request-limit-burst value, --visitor_request_limit_burst value                                               initial limit of requests per visitor (default: 60) [$NTFY_VISITOR_REQUEST_LIMIT_BURST]
   --visitor-request-limit-replenish value, --visitor_request_limit_replenish value                                       interval at which burst limit is replenished (one per x) (default: "5s") [$NTFY_VISITOR_REQUEST_LIMIT_REPLENISH]
   --visitor-request-limit-exempt-hosts value, --visitor_request_limit_exempt_hosts value                                 hostnames and/or IP addresses of hosts that will be exempt from the visitor request limit [$NTFY_VISITOR_REQUEST_LIMIT_EXEMPT_HOSTS]
   --visitor-message-daily-limit value, --visitor_message_daily_limit value                                               max messages per visitor per day, derived from request limit if unset (default: 0) [$NTFY_VISITOR_MESSAGE_DAILY_LIMIT]
   --visitor-email-limit-burst value, --visitor_email_limit_burst value                                                   initial limit of e-mails per visitor (default: 16) [$NTFY_VISITOR_EMAIL_LIMIT_BURST]
   --visitor-email-limit-replenish value, --visitor_email_limit_replenish value                                           interval at which burst limit is replenished (one per x) (default: "1h") [$NTFY_VISITOR_EMAIL_LIMIT_REPLENISH]
   --visitor-prefix-bits-ipv4 value, --visitor_prefix_bits_ipv4 value                                                     number of bits of the IPv4 address to use for rate limiting (default: 32, full address) (default: 32) [$NTFY_VISITOR_PREFIX_BITS_IPV4]
   --visitor-prefix-bits-ipv6 value, --visitor_prefix_bits_ipv6 value                                                     number of bits of the IPv6 address to use for rate limiting (default: 64, /64 subnet) (default: 64) [$NTFY_VISITOR_PREFIX_BITS_IPV6]
   --behind-proxy, --behind_proxy, -P                                                                                     if set, use forwarded header (e.g. X-Forwarded-For, X-Client-IP) to determine visitor IP address (for rate limiting) (default: false) [$NTFY_BEHIND_PROXY]
   --proxy-forwarded-header value, --proxy_forwarded_header value                                                         use specified header to determine visitor IP address (for rate limiting) (default: "X-Forwarded-For") [$NTFY_PROXY_FORWARDED_HEADER]
   --proxy-trusted-hosts value, --proxy_trusted_hosts value                                                               comma-separated list of trusted IP addresses, hosts, or CIDRs to remove from forwarded header [$NTFY_PROXY_TRUSTED_HOSTS]
   --stripe-secret-key value, --stripe_secret_key value                                                                   key used for the Stripe API communication, this enables payments [$NTFY_STRIPE_SECRET_KEY]
   --stripe-webhook-key value, --stripe_webhook_key value                                                                 key required to validate the authenticity of incoming webhooks from Stripe [$NTFY_STRIPE_WEBHOOK_KEY]
   --billing-contact value, --billing_contact value                                                                       e-mail or website to display in upgrade dialog (only if payments are enabled) [$NTFY_BILLING_CONTACT]
   --enable-metrics, --enable_metrics                                                                                     if set, Prometheus metrics are exposed via the /metrics endpoint (default: false) [$NTFY_ENABLE_METRICS]
   --metrics-listen-http value, --metrics_listen_http value                                                               ip:port used to expose the metrics endpoint (implicitly enables metrics) [$NTFY_METRICS_LISTEN_HTTP]
   --profile-listen-http value, --profile_listen_http value                                                               ip:port used to expose the profiling endpoints (implicitly enables profiling) [$NTFY_PROFILE_LISTEN_HTTP]
   --web-push-public-key value, --web_push_public_key value                                                               public key used for web push notifications [$NTFY_WEB_PUSH_PUBLIC_KEY]
   --web-push-private-key value, --web_push_private_key value                                                             private key used for web push notifications [$NTFY_WEB_PUSH_PRIVATE_KEY]
   --web-push-file value, --web_push_file value                                                                           file used to store web push subscriptions [$NTFY_WEB_PUSH_FILE]
   --web-push-email-address value, --web_push_email_address value                                                         e-mail address of sender, required to use browser push services [$NTFY_WEB_PUSH_EMAIL_ADDRESS]
   --web-push-startup-queries value, --web_push_startup_queries value                                                     queries run when the web push database is initialized [$NTFY_WEB_PUSH_STARTUP_QUERIES]
   --web-push-expiry-duration value, --web_push_expiry_duration value                                                     automatically expire unused subscriptions after this time (default: "60d") [$NTFY_WEB_PUSH_EXPIRY_DURATION]
   --web-push-expiry-warning-duration value, --web_push_expiry_warning_duration value                                     send web push warning notification after this time before expiring unused subscriptions (default: "55d") [$NTFY_WEB_PUSH_EXPIRY_WARNING_DURATION]
   --help, -h 
```
