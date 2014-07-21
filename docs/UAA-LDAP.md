==================================================
User Account and Authentication LDAP Integration
==================================================

.. contents:: Table of Contents

# Overview
UAA integrates with the Lightweight Directory Access Protocol in two major areas. 
Authentication, the first integration point, can be done using three different 
authentication methods.
Once authenticated, a users LDAP groups can be retrieved and mapped to scopes, 
the second integration point.

In this document we use the term 'bind' a lot and it refers to the LDAP
[bind operation](http://tools.ietf.org/html/rfc4511#section-4.2).
In short, it is the LDAP way of performing an authentication on a given connection 
to the LDAP server.

# Authentication

## Chained Authentication
When integrating with an external identity provider, such as LDAP,
authentication within the UAA becomes chained. An
authentication attempt with a user's credentials is first 
attempted against the UAA user store before the external provider, LDAP.

Chained authentication allows a certain number of boot strap users to 
exists within the UAA itself without the need to configure them in a potential
read only external store. 

Username's are not unique within the UAA. The combination of a username and 
it's origin, 'ldap' for example, is unique.

A potential collision does exist in a chained authentication. 
If the exact same set of credentials, username/password combination, exist in both the UAA and the LDAP 
server, a user would always be authenticated against the UAA and an LDAP authentication would not be attempted.
To avoid such a collision, and effectively disable chained authentication, 
do not bootstrap or create users in the UAA directly. 

## UAA Authentication
Authentication against the UAA database is the first step in the chained authentication.
Authentication is done using three variables

 * username - case insensitive username search, often email address
 * origin - matched against ['uaa'](https://github.com/cloudfoundry/uaa/blob/develop/common/src/main/java/org/cloudfoundry/identity/uaa/authentication/Origin.java#L16-16)
 * password - an encoded password, the input is encoded and then matched against the DB value
 
This is how the UAA performs user authentication ouf ot the box with no external
identity provider configured. When enabling LDAP, this method of authentication is always 
tried prior to attempting to authenticate against the external provider.

## LDAP Authentication
LDAP authentication within the UAA is done leveraging the 
[Spring Security LDAP module](https://github.com/spring-projects/spring-security/tree/master/ldap)
thus the authentication methods and configuration options you will find available within the UAA
are directly correlated to those found in Spring Security LDAP.
There are three different authentication methods supported against an LDAP compatible server

* Search and Bind - find the user DN, authenticate as the user
* Simple Bind - construct the user DN, authenticate as the user
* Search and Compare - find the user DN, perform a comparison against the users password attribute

### Ldap Search and Bind
The most common LDAP authentication method is the 'search and bind'.
During a search and bind

1. the user inputs a username
2. the UAA server performs a search using an LDAP filter and a set of known search credentials
3. If there is exactly one match the user's DN will be retrieved. Zero or more than one matches are automatically rejected.
4. The user's DN and supplied password is then used to attempt a bind against the LDAP server
5. The LDAP server performs the authentication

This method allows the most flexibility, as username to DN match is done using an LDAP search filter (query), 
and the user's credentials are not exposed. 

The security consideration with this authentication method is that a set of, preferably read only,
credentials has to be made available to the UAA to perform a search for a sub set of the 
LDAP tree where the user resides. Should the UAA's configuration be compromised,
so becomes the LDAP read only data that the UAA can query.

### Ldap Bind
An LDAP bind authentication never exposes any of the LDAP directory contents to the UAA. This is the benefit for a 
a less flexible authentication method.
In this method, the user supplies a username, and with that username, the UAA statically constructs a DN.
For example 

1. user supplies user, for example, filip - we construct DN, dn=uid=filip,ou=users,dc=test,dc=com
2. The UAA uses the statically constructed DN, and supplied password and attempts a bind to the LDAP server

In the simple bind method, the administrator of the UAA can configure one or more DN patterns
to be tried for a single username input. A pattern can also be the input itself, such as '{0}'
as the pattern means the user would have to type in his or her DN as the username,

While this method provides the most secure installation as no LDAP credentials or data
is exposed through configuration, it is not very common as the flexibility is reduced by not being able
to customize the username through a search query.

### Ldap Search and Compare
Similar to the search and bind, the search and compare will perform a search using a filter, retrieve the user's LDAP record, 
including the password field. It will then perform a password comparison against the password field in a similar way
that the UAA does against its user store.

This method is rarely used, it requires additional privileges to retrieve the password field.

## Ldap Authentication Configuration

### Overview
Configuration of the UAA is done through the [uaa.yml](https://github.com/cloudfoundry/uaa/blob/master/uaa/src/main/resources/uaa.yml)
This allows easy configuration for consumers like Cloud Foundry to generate a configuration file, and deploy the UAA as a job.

The UAA is a Spring based application, and reads the values from the uaa.yml and performs a variable substitution in the 
[XML configuration files](https://github.com/cloudfoundry/uaa/tree/master/uaa/src/main/webapp/WEB-INF/spring)

Enabling any level of LDAP authentication requires the 
[`spring_profiles: ldap`](https://github.com/cloudfoundry/uaa/blob/master/uaa/src/main/resources/uaa.yml#L7-7) configuration
to be enabled. This `ldap` profile triggers the chained authentication to be enabled and the 
[ldap configuration files](https://github.com/cloudfoundry/uaa/blob/develop/uaa/src/main/webapp/WEB-INF/spring-servlet.xml#L174)
to be loaded. 

<code>
spring_profiles: ldap
</code>

All further configurations will be placed under the `ldap: ` configuration element

#### Selecting an authentication method
Selecting an authentication method, `simple bind`, `search and bind` or `search and compare` is done using the 
`ldap.profiles.file` configuration attribute. There are three different values for this attribute, each mapped to the 
different authentication methods

* [`ldap/ldap-simple-bind.xml`](https://github.com/cloudfoundry/uaa/blob/develop/uaa/src/main/webapp/WEB-INF/spring/ldap/ldap-simple-bind.xml) - simple bind
* [`ldap/ldap-search-and-bind.xml`](https://github.com/cloudfoundry/uaa/blob/develop/uaa/src/main/webapp/WEB-INF/spring/ldap/ldap-search-and-bind.xml) - search and bind
* [`ldap/ldap-search-and-compare.xml`](https://github.com/cloudfoundry/uaa/blob/develop/uaa/src/main/webapp/WEB-INF/spring/ldap/ldap-search-and-compare.xml) - search and compare

As noticed, the attribute is an actual reference to a configuration file. The configuration, 
at a minimum, should provide a bean named `ldapAuthProvider` that will be used
to configure the 
[LDAP authentication manager](https://github.com/cloudfoundry/uaa/blob/develop/uaa/src/main/webapp/WEB-INF/spring/ldap-integration.xml#L44).

This allows a user/administrator of the UAA to configure a Spring XML file for a custom ldap authentication method.

<pre>
spring_profiles: ldap
ldap:
  profile:
    file: ldap/ldap-search-and-bind.xml
</pre>

##### Configuring Simple Bind
The following attributes are available for the default bind configuration

* `ldap.base.url` - A URL pointing to the LDAP server, must start with `ldap://` or `ldaps://`
* `ldap.base.userDnPattern` - one or more patterns used to construct DN.

<pre>
spring_profiles: ldap
ldap:
  profile:
    file: ldap/ldap-simple-bind.xml
  base:
    userDnPattern: 'cn={0},ou=Users,dc=test,dc=com;cn={0},ou=OtherUsers,dc=example,dc=com'
</pre>


##### Configuring Search and Bind
The following attributes are available for the default search and bind configuration

* `ldap.base.url` - A URL pointing to the LDAP server, must start with `ldap://` or `ldaps://`
  In the case of SSL (ldaps), the server must hold a trusted certificate or the certificate must be
  imported into the JVM's truststore. 
* `ldap.base.userDn` - The DN for the LDAP credentials used to search the directory
* `ldap.base.password` - Password credentials for the above DN to search the directory
* `ldap.base.searchBase` - Specify only if a part of the directory should be searched, for example
  `dc=test,dc=com`
* `ldap.base.searchFilter` - the search filter used for the query. `{0}` is used to annotate 
  where the username will be inserted. For example `cn={0}` will search the LDAP directory records
  where the attribute `cn` matches the users input.

<pre>
spring_profiles: ldap
ldap:
  profile:
    file: ldap/ldap-search-and-bind.xml
  base:
    url: 'ldap://localhost:10389/'
    userDn: 'cn=admin,ou=Users,dc=test,dc=com'
    password: 'password'
    searchBase: ''
    searchFilter: 'cn={0}'
</pre>

##### Configuring Search and Compare
The following attributes are available for the default search and bind configuration

* `ldap.base.url` - A URL pointing to the LDAP server, must start with `ldap://` or `ldaps://`
  In the case of SSL (ldaps), the server must hold a trusted certificate or the certificate must be
  imported into the JVM's truststore. 
* `ldap.base.userDn` - The DN for the LDAP credentials used to search the directory
* `ldap.base.password` - Password credentials for the above DN to search the directory
* `ldap.base.searchBase` - Specify only if a part of the directory should be searched, for example
  `dc=test,dc=com`
* `ldap.base.searchFilter` - the search filter used for the query. `{0}` is used to annotate 
  where the username will be inserted. For example `cn={0}` will search the LDAP directory records
  where the attribute `cn` matches the users input.
* `ldap.base.passwordAttributeName` - the name of the LDAP attribute that holds the password
* `ldap.base.localPasswordCompare` - set to true if the comparison should be done locally
  Setting this value to false, implies that rather than retrieving the password, the UAA
  will run a query to match the password. In order for this query to work, you must know what 
  type of hash/encoding/salt is used for the LDAP password.
* `ldap.base.passwordEncoder` - A fully qualified Java classname to a password encoder.
  The [default](https://github.com/cloudfoundry/uaa/blob/master/common/src/main/java/org/cloudfoundry/identity/uaa/ldap/DynamicPasswordComparator.java#L20-20)
  uses the Apache Directory Server password utilities to support several different encodings.

<pre>
spring_profiles: ldap
ldap:
  profile:
    file: ldap/ldap-search-and-bind.xml
  base:
    url: 'ldap://localhost:10389/'
    userDn: 'cn=admin,ou=Users,dc=test,dc=com'
    password: 'password'
    searchBase: ''
    searchFilter: 'cn={0}'
    passwordAttributeName: userPassword
    passwordEncoder: org.cloudfoundry.identity.uaa.login.ldap.DynamicPasswordComparator
    localPasswordCompare: true
</pre>

# LDAP Group Mapping

As of now, the primary purpose of the UAA is to issue [Oauth 2](http://tools.ietf.org/html/rfc6749) 
tokens to the client on behalf of the user. 

The UAA integrates with LDAP groups during the user authentication process. Each time a user is authenticated,
group memberships, if configured, are retrieved and refreshed.

These groups are then mapped to UAA scopes.

## Scopes

A token contains a list of scopes and authorities. The scopes in the token represent the permissions of the user
while the authorities represent the permissions of the client itself. Thus, a resource server receiving a request
containing a token can decide to authorize the client on behalf of the user, or just the client itself.



## Ldap Groups as Scopes

## Ldap Groups to Scopes

### Populating External Group Mappings

## Ldap Group to Scope Configuration

# Samples

# References

