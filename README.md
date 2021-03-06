> **WARNING: This plugin requires Spring Security Core 2.0-RC2 or later to work properly.  It _WILL NOT_ work with older
> versions due to dependency conflicts.**

This plugin is an OAuth2 Provider based on the Spring Security OAuth libraries.  It is partially based off of Burt Beckwith's OAuth
Provider plugin, which was never officially released.

While this plugin works for certain use cases, not all OAuth2 flows have been tested.  In particular, the following works and has
been tested:

* The full flow of logging in with both users and clients using tokens and authorization codes

However, the following items have not been tested and may or may not work:

* Grant types besides `authorization_code` and `client_credentials`
* Protected resources via Spring OAuth2 - this is currently done with the Spring Security core methods (ie request maps, annotations, intercept maps)

## Setup

> You MUST follow the steps below in order to get the plugin working correctly.

### Configure Security Rules

The `authorizationEndpointUrl` and the `tokenEndpointUrl` must both be protected by the Spring Security Core plugin.
The configuration below will accomplish this using static rules in Config.groovy:

```groovy
grails.plugin.springsecurity.controllerAnnotations.staticRules = [
	'/oauth/authorize.dispatch':['IS_AUTHENTICATED_REMEMBERED'],
	'/oauth/token.dispatch':['IS_AUTHENTICATED_REMEMBERED'],
]
```

> Note that the URLs are mapped with `.dispatch` at the end.  This is essential in order to correctly protect the resources.  For
> example, an `authorizationEndpointUrl` of `/custom/authorize-oauth2` would need to be protected with `/custom/authorize-oauth2.dispatch`.

### Add Client Provider

Second, add the `clientCredentialsAuthenticationProvider` name to the `grails.plugin.springsecurity.providerNames` list.
This is required in order to use the authentication/authorization flows listed below.

```groovy
// Add to the default list from spring security core
grails.plugin.springsecurity.providerNames = [
		'daoAuthenticationProvider',
		'anonymousAuthenticationProvider',
		'rememberMeAuthenticationProvider',
		'clientCredentialsAuthenticationProvider'
]
```

### (Optional) Customize Confirm View

On install, a view is created at `grails-app/views/oauth/confirm.gsp`.  This view may be modified as desired, but the
location should match the `userApprovalEndpointUrl` setting discussed below.

## How to Use

### Register Clients in Configuration

Clients are configured using the default in memory client details service in an option in Config.groovy or an external configuration
file specified by `grails.config.locations`.  The following is an example of configuring a simple client with an ID
of `myId` and a secret key of `mySecret`:

```groovy
grails.plugin.springsecurity.oauthProvider.clients = [
	[
		clientId:"myId",
		clientSecret:"mySecret"
	]
]
```

Notice that the client configuration consists of a list of maps with each map representing a single configured client.
The properties which can be configured match the properties in the `org.springframework.security.oauth2.provider.BaseClientDetails`
class.  The only difference is the `webServerRedirectUri` has been renamed to `registeredRedirectUri` in order to be compatible
with newer releases of Spring Security OAuth2.  Default values have been configured for each property except for `clientId` and
`clientSecret` since these are unique for each configured client.  These default values are shown in the following code block with
their default values.  These default values may be modified by placing a line similar to the following in Config.groovy or an external
configuration file.

```groovy
grails.plugin.springsecurity.oauthProvider.defaultClientConfig.resourceIds = []
grails.plugin.springsecurity.oauthProvider.defaultClientConfig.authorizedGrantTypes = ["authorization_code", "refresh_token"]
grails.plugin.springsecurity.oauthProvider.defaultClientConfig.scope = []
grails.plugin.springsecurity.oauthProvider.defaultClientConfig.registeredRedirectUri = null
grails.plugin.springsecurity.oauthProvider.defaultClientConfig.authorities = []
grails.plugin.springsecurity.oauthProvider.defaultClientConfig.accessTokenValiditySeconds = []
grails.plugin.springsecurity.oauthProvider.defaultClientConfig.refreshTokenValiditySeconds = []
```

For example, with a default configuration option in Config.groovy of:

```groovy
grails.plugin.springsecurity.oauthProvider.defaultClientConfig.authorizedGrantTypes = ["implicit"]
```

And a client configuration of:

```groovy
grails.plugin.springsecurity.oauthProvider.clients = [
	[
		clientId:"myId",
		clientSecret:"mySecret"
	]
]
```

Will result in a client with an ID of `myId` and a single authorized grant type of `implicit`.  However, if the client configuration
was modified to the following:

```groovy
grails.plugin.springsecurity.oauthProvider.clients = [
	[
		clientId:"myId",
		clientSecret:"mySecret",
		authorizedGrantTypes:["authorization_code"]
	]
]
```

Then the resulting configured client `myId` would have a single authorized grant type of `authorization_code`.  In other words,
the default configuration is overridden by the individual client configuration.

Using this method, when the configuration changes, the entire list of clients is reloaded and replaces the old list.

### Registering Clients in Bootstrap

Clients may also be individually registered by using the `BaseClientDetails` class in combination with the `clientDetailsService`
bean in the Bootstrap process.  The following is an example of registering a client programmatically (to be run in the BootStrap of your application):

```groovy
import org.springframework.security.oauth2.provider.BaseClientDetails;

class BootStrap {
	def clientDetailsService

	def init = { servletContext ->
		def client = new BaseClientDetails()
		client.clientId = "clientId"
		client.clientSecret = "clientSecret"
		client.authorizedGrantTypes = ["authorization_code", "refresh_token", "client_credentials", "password", "implicit"]

		// Set the full contents of the client details service store to the newly created client
		clientDetailsService.clientDetailsStore = [
			// Map of client ID to the client details instance
			"clientId":client
		]
	}
```

## Examples

The [examples](examples) directory may be very useful in implementing the flows below.  Especially note the
[Scribe User Approval Script](examples/user-approval-scribe.groovy) for an example of the User Approval flow.

## Login Flows

### Client Login

The client may login with the URL given in the `tokenEndpointUrl` setting (`/oauth/token` by default) by using the following syntax.
Notice the `grant_type` of `client_credentials` and that the client credentials from the example above are used.

```
http://localhost:8080/app/oauth/token?grant_type=client_credentials&client_id=clientId&client_secret=clientSecret
```

The response from a login such as this is the following JSON.  The `access_token` may then be used to access resources while the refresh token may be used at a later time to get a new access token (see section on refresh tokens below for more information).

```javascript
{
  "access_token": "449acfe6-663f-4fde-b1f8-414c867a4cb5",
  "expires_in": 43200,
  "refresh_token": "ab12ce7a-de9d-48db-a674-0044897074b0"
}
```

### User Approval of Clients

The following URLs or configuration options show a typical flow authorizing a client for a certain user.

* The user must be logged into the application protected by this plugin.  Alternatively, they will be logged in
on the next step since the `authorizationEndpointUrl` must be protected with Spring Security Core as noted above.
* A client attempting to use a service provided by the OAuth protected application is reached by a user.  The
client then redirects the user to the `authorizationEndpointUrl` setting (`/oauth/authorize` by default).  This will actually
redirect the user to the `userApprovalEndpointUrl` setting which will present the user with an option to authorize or deny access
to the application for the client.

```
http://localhost:8080/app/oauth/authorize?response_type=code&client_id=clientId&redirect_uri=http://localhost:8080/app/
```

The user will then be redirected to the `redirect_uri` with the code appended as a URL parameter such as:

```
http://localhost:8080/app/?code=YjZOa8
```

* The client captures this code and sends it to the application at the `tokenEndpointUrl` setting.
This will allow the client to access the application as the user.  Notice the `grant_type` of `authorization_code` this time.

```
http://localhost:8080/app/oauth/token?grant_type=authorization_code&client_id=clientId&client_secret=clientSecret&code=OVD8SZ&redirect_uri=http://localhost:8080/app/
```

This will then give an access token and a refresh token to the client along with an expiration date for the access token.  The access token is used to access the application as the user, while the refresh token can be used to obtain another access token (see section on refresh tokens below for more information).  In other words, the access token is a temporary grant to the resources the user can access, while the refresh token is a permanent token that may be used to update the access token.

```
{
  "access_token":"4b3f8efb-c919-4bcf-a430-266e5cede8e9",
  "token_type":"bearer",
  "refresh_token":"1333e0d5-f885-4f71-86ea-f90a081b1a21",
  "expires_in":43199
}
```

> WARNING: The redirect_uri in the `code` response and the `authorization_code` grant must match!  Otherwise, the authorization will fail.

### Refreshing Tokens

After receiving the access and refresh tokens, the client may optionally be granted a new access token using the refresh token combined with their client ID and secret.  They simply need to go to the following URL:

```
http://localhost:8080/app/oauth/token?grant_type=refresh_token&client_id=clientId&client_secret=clientSecret&refresh_token=1333e0d5-f885-4f71-86ea-f90a081b1a21&redirect_uri=http://localhost:8080/app/
```

This will give a response similar to the first grant of the tokens but will contain a new access token:

```
{
  "access_token":"3d6f4bd6-48ea-4512-bf02-9724a23f7882",
  "token_type":"bearer",
  "refresh_token":"1333e0d5-f885-4f71-86ea-f90a081b1a21",
  "expires_in":43199
}
```

### Protecting Resources

If the instructions above are followed, this plugin will provide access to resources protected with the `Secured` annotation or with
static rules defined in Config.groovy.  Resources protected with request maps or other spring security configurations *should* be protected,
but is untested.  If you have tested this plugin in these configurations, please let me know and I'll update this section.

## Configuration

### Endpoint URLs

By default, three endpoint URLs have been defined.  Note that default URLMappings are provided for the
`authorizationEndpointUrl` and the `tokenEndpointUrl`.  If these are modified, additional URLMappings will have to
be set.  Their default values and how they would be set in Config.groovy are shown below:

```groovy
grails.plugin.springsecurity.oauthProvider.authorizationEndpointUrl = "/oauth/authorize"
grails.plugin.springsecurity.oauthProvider.tokenEndpointUrl = "/oauth/token"	// Where the client is authorized
grails.plugin.springsecurity.oauthProvider.userApprovalEndpointUrl = "/oauth/confirm"	// Where the user confirms that they approve the client
```

NOTE: The `userApprovalEndpointUrl` never is actually redirected to, but is simply used to specify the location of the view.
For example, a `userApprovalEndpointUrl` of `/custom/oauth_confirm` would map to the `grails-app/views/custom/oauth_confirm.gsp` view.

### Grant Types

The grant types for OAuth authentication may be enabled or disabled with simple configuration options.  By default all grant types are enabled.
Set the option to false to disable it completely, regardless of client configuration.

```groovy
grails.plugin.springsecurity.oauthProvider.grantTypes.authorizationCode = true
grails.plugin.springsecurity.oauthProvider.grantTypes.implicit = true
grails.plugin.springsecurity.oauthProvider.grantTypes.refreshToken = true
grails.plugin.springsecurity.oauthProvider.grantTypes.clientCredentials = true
grails.plugin.springsecurity.oauthProvider.grantTypes.password = true
```

### Configuration

Here are some other configuration options that can be set and their default values.  Again, these would be placed in Config.groovy:

```groovy
grails.plugin.springsecurity.oauthProvider.active = true // Set to false to disable the provider, true in all environments but test where false is the default
grails.plugin.springsecurity.oauthProvider.filterStartPosition = SecurityFilterPosition.X509_FILTER.order // The starting location of the filters registered
grails.plugin.springsecurity.oauthProvider.clientFilterStartPosition = SecurityFilterPosition.DIGEST_AUTH_FILTER.order // The starting location of the client filters registered (should be before the basic auth filter)
grails.plugin.springsecurity.oauthProvider.userApprovalParameterName = "user_oauth_approval" // Used on the user confirmation page (see userApprovalEndpointUrl)
grails.plugin.springsecurity.oauthProvider.tokenServices.refreshTokenValiditySeconds = 60 * 10 //default 10 minutes
grails.plugin.springsecurity.oauthProvider.tokenServices.accessTokenValiditySeconds = 60 * 60 * 12 //default 12 hours
grails.plugin.springsecurity.oauthProvider.tokenServices.reuseRefreshToken = true
grails.plugin.springsecurity.oauthProvider.tokenServices.supportRefreshToken = true
```

## Release Notes

### 1.0.5.2

* Fix #13 - Make clientSecret optional in client configuration structure

### 1.0.5.1

* Merge pull request #21 (Burt's cleanup)
* Use log wrapper instead of log4j
* Depends on Grails 2.0 or greater (consistent with core plugin)

### 1.0.5

* Initial release of plugin compatible with spring security core 2.0-RC2
