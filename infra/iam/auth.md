# Deep dive into Authorization on GCP

> This document is about how actors using GCP resources are authorized to perform GCP API calls, not about how applications hosted on GCP can authenticate end-users.

> This document is not about IAM or about security on GCP in general.

## Basics
Let's get a few basics right first.

* **Authentication** is checking the _identity_ of the calling party, like when _logging in_
* **Authorization** is checking if the caller should be allowed to access a specific resource or perform a specific operation. This optionally involves authentication.

In a traditional system, authorization is bound to authentication. The general flow is that the system identifies the calling party, then it checks Access Control Lists to check if the operation or access should be allowed or not. Most systems work with users, roles and permissions, optionally also involving groups and resource-bound permissions.

### OAuth2
OAuth2 provides the basis for authorization in GCP, so it's worth taking time to get to know it well. I also encourage you to spend time reading the original [RFCs](https://datatracker.ietf.org/doc/html/rfc6749) (there are various RFCs involved).

You are also encouraged to review other online materials that give you an introduction to the topic. Nevertheless, I'll also cover some important basics below.

Some trivia:
* OAuth2 is only about authorization, it's not about identifying the calling identity
* OpenID extends OAuth2 to enable accessing identity information
* An OpenID/OAuth2 provider isn't necessarily a user directory, the specifications don't mandate that there's a list of users and groups.
* OpenID/OAuth2 don't enforce anything about how the providers _authenticate_ and manage their users.
* OAuth2 is not compatible with OAuth1, it's a completely different standard. OAuth1 is outdated and not really used anymore.

### Terminology

Important OAuth2 terminology:
* A **resource owner** is usually the end user who is trying to act on a resource.
* A **user agent** is basically the browser.
* The **resource server** holds the resources and receives operation requests.
* A **client** is an application that the resource owner uses to act on the resource server.
* The **authorization server** is used for auth operations.
server
For example, you as a resource owner can act in the browser as the user agent to perform a compute engine instance creation. You can use the API Explorer for this, in this case the authorization servers are Google servers, the resource servers are compute engine api servers, and the client is the API Explorer.

Note that OAuth2 is often used to provide __authentication__ to 3rd party services. This involves elements from the OpenID standards (covered later). In this flow, the external site, eg `mycutekittenpictures.meow` registers a _client_ with the Google Auth servers, and when you log in, you _authorize_ this client to request your identity information from _token info_ endpoints.

OAuth2 itself doesn't prescribe how the authorization info is sent from the user agent to the resource server. The generally applied solution is [_Bearer tokens_](https://datatracker.ietf.org/doc/html/rfc6750).

Also, OAuth2 itself doesn't prescribe how the resource server can validate the authorization tokens. [RFC7662](https://datatracker.ietf.org/doc/html/rfc7662) specifies an _introspection endpoint_, but this is not implemented by Google OAuth for example. Other solutions can involve shared databases or encoding information in the tokens themselves.

### Flows

Obtaining an access token can be performed in various ways, these are called _authorization flows_. You've most probably seen these various times.
* The **Authorization code flow** is commonly used for web applications with server-side components. The browser is redirected to the auth servers, it asks you for confirmation, then it redirects you back to the application servers with an authorization code, and the app servers then exchange this code to an access token on the server-side. The server can then use the access token to make API calls.
* The **Implicit grant flow** is widely used in case of Single-Page Applications, for example with the popular `gapi` library. In this case there's no app server-side and no authorization code involved, the access token is returned directly in the redirect url.

Note that in the authentication code flow, the app server side has to somehow authorize its 'token exchange' call to the authorization servers. On GCP this is done by using a 'client id' and a 'client secret', but this is not specified in related standards.

### Scopes
OAuth2 authorizes you for _scopes_. Scopes are generally opaque strings, their usage is not specified in any ways by OAuth2, but the scopes are often included in authorization URLs or JWT tokens, so there should be sufficiently few and small scope strings.

* It can be a specific piece of information, like https://www.googleapis.com/auth/userinfo.email 
* A specific kind of resources / a specific group of APIs: https://www.googleapis.com/auth/compute
* Restricted operations on these APIs: https://www.googleapis.com/auth/compute.readonly
* A broader union of stuff: https://www.googleapis.com/auth/cloud-platform
* Or any kind of custom wild logic: https://www.googleapis.com/auth/drive.file

### OpenID Connect
For our purposes, OpenID Connect or OIDC is important because it provides _identity information_ as part of the OAuth2 mechanisms. OIDC is otherwise a much broader range of standards.

### Quick example
An example of an authentication code flow.

You can use "APIs and Services" / "Credentials", then "Create Credentials" / "OAuth2 Client ID" to create a client.

To initiate a flow, open a URL in the browser:
```
https://accounts.google.com/o/oauth2/v2/auth
?response_type=code
&client_id=123456789-xxxx.apps.googleusercontent.com
&redirect_uri=http%3A%2F%2Flocalhost%3A8080%2Fcallback
&scope=openid%20profile%20email
&state=ABC123
```
This contains the auth URL specified for Google. The `client id` is the client we've registered. The `response_type` specifies which flow we'd like to use. The `scope` specifies the things we'd like to have authorizations for. Parameters `state` and `redirect_uri` are checked for security.

This page will prompt the user to grant authorization for our client for the requested scopes. If we aren't logged in, it will require us to log in (note that the way this happens is not specified by OAuth).

Google will then redirect the browser back to the 'callback url' specified for the client. The server side can then use this piece of code to exchange tokens:

```javascript
    const tokens = await axios.post("https://oauth2.googleapis.com/token", {
        "grant_type": "authorization_code",
        "code": req.query.code,
        "redirect_uri": "http://localhost:8080/callback",
        "client_id": "123456789-xxxx.apps.googleusercontent.com",
        "client_secret": client_secret
    })
    const jwt = jwt_decode(tokens.data.id_token);
```

A typical response to this request will contain:
```json
{
    "access_token": "ya29.xxxx",
    "expires_in": 3599,
    "scope": "....",
    "token_type": "Bearer",
    "id_token": "..."
}
```

You can then use this access token to make API calls on GCP:

```javascript
    const userInfo = await axios.get("https://openidconnect.googleapis.com/v1/userinfo", { headers: {
        "Authorization": "Bearer " + tokens.data.access_token
    }})
```

The `id_token` originally returned is an encoded JWT token that also contains identity information, like `email`, `picture` and `sub`. The usage of `id_token` is specified in the OIDC standard.

### Refresh tokens
Access tokens are usually short-lived, by default they are valid for an hour. In specific cases OAuth providers can return _refresh tokens_ besides access tokens in the response shown above.

The application servers can then use these refresh tokens to get new access tokens without user interaction. This is especially useful for applications that require 'background' API calls for their operation, for example regularly synchronizing calendar entries.

Refresh tokens themselves don't expire. Client authorizations can be revoked in your accounts security page - this part is of course not specified in standards.

### JWT
JWT, or [JSON Web Tokens](https://jwt.io/introduction) are a web-friendly data format to transfer identity and authorization information.

Its terminology works with _claims_, in the sense that the sender of the token _states_ a set of data.

An encoded JWT token contains a header, payload and a signature. This way the receiver of the token can verify the contents of the token against the signature. The signature uses an asymmetric key pair. The authorization servers signing JWT tokens have a published public key that token receivers can use to verify the signature.

In our use cases, the signature ensures not only that the transferred data is coherent. A signature on a JWT token also means that the OIDC provider of the identity was issued to an authorized client, meaning that the holder of the signed JWT token is authorized to act as the subject of the token with the given scopes.

To see a JWT token, you can use:

```> gcloud auth print-identity-token```

You can then safely copy-paste this token to [https://jwt.io](https://jwt.io) and see the information it contains.

Most typically it will contain:
```json
{
  "iss": "https://accounts.google.com",
  "aud": "32555940559.apps.googleusercontent.com",
  "sub": "xxx",
  "email": "gabor.farkas@aliz.ai",
  "iat": 1647537566,
  "exp": 1647541166
}
```

* **iss** for issuer
* **aud** for audience, the client id for which this token was issued
* **sub** the id of the subject
* **email** optionally, the email address of the subject
* **iaut** issued at timestamp
* **exp** expires at timestamp

## Google Cloud Platform
And now let's see how all this works on GCP.

Some useful material:
* [https://cloud.google.com/docs/authentication](https://cloud.google.com/docs/authentication)
* [https://developers.google.com/identity/protocols/oauth2](https://developers.google.com/identity/protocols/oauth2)
* [https://developers.google.com/identity/protocols/oauth2/openid-connect](https://developers.google.com/identity/protocols/oauth2/openid-connect)

Basically it will be OAuth2 / OIDC in the end.

### Actors and clients
When a business decides to use Google Cloud, there are basically two options:
* Use Google Workspace. Most typically the business already has a Workspace domain, or they can decide to register one. In this case, Google Workspace will provide the user and group directory
* Synchronize users and groups from Active Directory.

The service providing identities for GCP is called _Cloud Identity_. This is not to be confused with _Identity Platform_ which is a service that can be integrated to applications to provide auth.

Actors on GCP will include:
* Users as defined by Cloud Identity
* Service Accounts, created by the customer
* Default Service Accounts are pretty much like custom Service Accounts, but they are automatically created when you start using certain services like Compute Engine or Cloud Build. They don't always support all features you can otherwise do with custom service accounts.
* Service Robots are also much like service accounts, but they support even less features.

Clients acting on GCP will include:
* The web console. Authorization here happens in a different way, we are not covering this now.
* The `gcloud` command line client uses good old OAuth2
* Client libraries
* Custom code

### API Explorer

For example, if you open up the [Compute Engine REST API](https://cloud.google.com/compute/docs/reference/rest/v1), you can make calls using the API Explorer on the right side. When you first make an API call, it will open an authorization popup.

The URL here should look familiar:
```
https://accounts.google.com/o/oauth2/auth/oauthchooseaccount
?response_type=permission%20id_token
&scope=https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fcloud-platform%20https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fcompute%20https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fcompute.readonly%20https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fuserinfo.email
&redirect_uri=storagerelay%3A%2F%2Fhttps%2Fexplorer.apis.google.com%3Fid%3Dauth792123
&client_id=292824132082.apps.googleusercontent.com
```

You can spot the client_id, the scopes, the redirect_uri and other familiar parameters. This uses the SPA-focused 'id_token' based flow, so the access token will be directly returned to be used in the browser. The client_id belongs to a registered application (client), named API Explorer.

### gcloud
If you use `gcloud auth login` or `gcloud auth application-default login`, it will also follow the same OAuth2 flow. These have their own respective client ids and default scopes.

These login flows will receive a refresh_token and will store these tokens in local files under your home directory. When you try to make an API call, these tokens will be used to get an access token that can be used to make the call.

You can try this manually:

```
curl -H "Authorization: Bearer $(gcloud auth print-access-token)"\
  https://openidconnect.googleapis.com/v1/userinfo
```

Here we use the `gcloud auth print-access-token` command to get an access token and then include it as a header to a simple `curl` call to get the user info as defined in OIDC.

You might be wondering where this userinfo endpoint URL comes from. There's a detailed description [here](https://developers.google.com/identity/protocols/oauth2/openid-connect) of how Google implements the OIDC standard.

The standard prescribes publishing the [.well-known/openid-configuration](https://accounts.google.com/.well-known/openid-configuration) file.

This contains for example:
```json
{
 "issuer": "https://accounts.google.com",
 "authorization_endpoint": "https://accounts.google.com/o/oauth2/v2/auth",
 "device_authorization_endpoint": "https://oauth2.googleapis.com/device/code",
 "token_endpoint": "https://oauth2.googleapis.com/token",
 "userinfo_endpoint": "https://openidconnect.googleapis.com/v1/userinfo",
 "revocation_endpoint": "https://oauth2.googleapis.com/revoke",
 "jwks_uri": "https://www.googleapis.com/oauth2/v3/certs"
}
```

### gcloud credentials
It's important to know that if you authenticate using `gcloud auth login`, that authentication will only affect the commands you make using the gcloud command-line tool itself. You can login with multiple accounts and switch between the authenticated users. Gcloud also supports some common parameters that provide additional features, for example impersonating service accounts.

### Client libraries
If you create an application, you can call Google APIs basically any way you'd like. The `curl` based approach shows an example above.

Most often you'll use client libraries maintained by Google in your language and environment of choice. Google maintains these libraries for the most popular languages, like Java, Go, .NET, Node.js, Python and so on.

When you initiate the client stub for a given API, you can specify the way you'd like to authenticate the requests to that API, though client library implementations differ in the variety of ways they provide.

You also have the choice of not specifying any authentication. In this case the client libraries will use the Application-Default Credentials flow.

### Application-Default Credentials
This is a 'recommendation' specified in [API-4110](https://google.aip.dev/auth/4110) and related entries, but all Google-maintained client libraries implement this flow.

If you initialize a client library without specifying an explicit authentication method, it will follow this mechanism.

Please refer to the linked specifications for the complete picture, but let me point out the most important parts and use-cases.

#### End-user login

First, you can use `gcloud auth application-default login` to initiate a regular OAuth2-based flow. This has a specific client id, and it has the `cloud-platform` and other OIDC related scopes by default, but you can also specify additional scopes. You can only log in as end-users with this mechanism.

#### Service account key file

To work on behalf of a service account, one option is to export a service account key file and store it on the target machine either at a prescribed location, or any other location and set the `GOOGLE_APPLICATION_CREDENTIALS` environment variable to that path.

It's important to note that exporting service account keys should be avoided if possible, as they impose a high security risk. We'll discuss some common practices later below.

A service account key file will basically contain a private key that the client library will use to sign a JWT token request. This signed and encoded JWT token will be sent as a header to a [`generateAccessToken`](https://cloud.google.com/iam/docs/reference/credentials/rest/v1/projects.serviceAccounts/generateAccessToken) API call, which will return the access_token that will then be used by the client library to make subsequent API calls. We'll cover more details about this mechanism later below.

#### Metadata server
Whenever your code runs in a GCP environment, Google will _usually_ provide a locally accessible _metadata server_, as specified in [AIP-4115](https://google.aip.dev/auth/4115).

You can try this on a GCP VM:

```
curl -H "Metadata-Flavor:Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token
```

This token will be the same as the one you get when using `gcloud auth application-default print-access-token`, but on a GCP VM, the regular gcloud auth will also work with this token.

As a side-note, the metadata server also provides some other information specific to environments.

Other environments like Cloud Run, Cloud Functions, Cloud Build or App Engine also provide this mechanism.

**Service Account** - the service account is configured when creating the VM itself, or when you configure the code-running service. If you 'just create' a new VM, the web console defaults to the project default compute service account, but you can select custom service accounts there, and it's actually encouraged to use specific service accounts instead of the default one.

You can also create a VM without an associated service account, in this case the metadata server will return an error, and gcloud will not work.

You can also manually log in using `gcloud auth login` on GCP VMs, but as the gcloud tool also warns you about it, this should be generally avoided.

**Scopes** - When you are creating the VM on the web console, you have an option to select which APIs you wish the VM to have access to. This setting finally translates to a set of scopes that the token will be authorized to.

There's a [gcloud beta command](https://cloud.google.com/sdk/gcloud/reference/beta/compute/instances/set-scopes) to set additional scopes on a VM. Other services don't yet support specifying additional scopes, but they default to the `cloud-platform` and other OIDC-related scopes, so you'll be able to access all GCP services this way.

Some exotic services like the App Engine Remote API, or non-GCP services like the Google Workspace Admin API can require further scopes.

You can also use the metadata server to retrieve the set of tokens your access token will have:

```
curl -H "Metadata-Flavor:Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/scopes
```

You can find a full reference of the accessible values [here](https://cloud.google.com/compute/docs/metadata/default-metadata-values). 

When working with **Google Kubernetes Engine**, the default case is that the pods running on the node VMs access the metadata server, so client libraries running on pods will default to use the identity of the service account associated with nodes. Note that GKE nodes must have an associated service account to let the kubelet communicate with the API server.

_Metadata concealment_ can prevent pods from accessing the metadata server, but the ultimate recommendation is to use _Workload Identity_. We'll cover this later, but in short this lets you associate specific, workload-related service accounts to specific deployments or pods.

#### Internals
It might be worth taking a look at how the  Application-Default Credentials flow is implemented in your client library of choice. For example:
* [NodeJS](https://github.com/googleapis/google-auth-library-nodejs/blob/097d3283a9c34da9ec9734079730e22bbd7daf6e/src/auth/googleauth.ts#L267)
* [Golang](https://github.com/golang/oauth2/blob/6242fa91716a9b0e1cbc7506d79c18c64a2341e8/google/default.go#L111)
* [Java](https://github.com/googleapis/google-api-java-client/blob/361e7fef50cb46b159873327f6f3d40ecb61a7ec/google-api-client/src/main/java/com/google/api/client/googleapis/auth/oauth2/DefaultCredentialProvider.java#L111)

You can also notice that there are 3 types of local credential files, see for example [here](https://github.com/googleapis/google-auth-library-nodejs/blob/097d3283a9c34da9ec9734079730e22bbd7daf6e/src/auth/googleauth.ts#L489). The local file is always a json with a mandatory differentiator field named `type`.

##### User credentials file
A user credentials file will look like this:
```
{
  "client_id": "764086051850-6qr4p6gpi6hn506pt8ejuq83di341hur.apps.googleusercontent.com",
  "client_secret": "xxx",
  "quota_project_id": "xxx,
  "refresh_token": "xxx",
  "type": "authorized_user"
} 
```
This stores the result of a regular OAuth2 authorization code flow, where the server-side component is the gcloud client itself. It stores the refresh token in this file and exchanges that for an access token when necessary. Needless to mention that you should protect these files properly.

##### Service account key
While it's generally discouraged, you can export a key for service accounts you created.

In the json format they look like this:
```
{
  "type": "service_account",
  "project_id": "project-id",
  "private_key_id": "0eb562fc24fb0c5066021b4d57a65",
  "private_key": "-----BEGIN PRIVATE KEY-----xxxx\n-----END PRIVATE KEY-----\n",
  "client_email": "email@address",
  "client_id": "10037409879297",
  "auth_uri": "https://accounts.google.com/o/oauth2/auth",
  "token_uri": "https://oauth2.googleapis.com/token",
  "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
  "client_x509_cert_url": "https://www.googleapis.com/robot/v1/metadata/x509/email%40address"
}
```

But it can also be exported as a P12 file only.

When you create such a key, Google generates a key pair actually, it keeps the public key part on its side to verify the signatures you make with this private key.

This private key will be used to [construct](https://github.com/googleapis/google-auth-library-nodejs/blob/097d3283a9c34da9ec9734079730e22bbd7daf6e/src/auth/jwtclient.ts#L186) a JWT token, which is then [sent](https://github.com/googleapis/google-auth-library-nodejs/blob/097d3283a9c34da9ec9734079730e22bbd7daf6e/src/auth/oauth2client.ts#L619) to the token exchange URL `https://oauth2.googleapis.com/token`. The response in this case will contain an access token.

You can also work manually with the exported P12, you can sign a JWT request the same way as in the code pieces above.

##### External Account
The external account type is used for Workload Identity Federation. We'll cover this in more detail later below.

### Making API Calls
We've already seen examples above, you basically need to provide the `Authorization` header with HTTP calls, including a `Bearer` token.

This bearer token is most commonly an access token. Some APIs also support receiving signed JWT tokens as Bearer tokens.

For an example using an access token:

```
curl -H "Authorization: Bearer $(gcloud auth application-default print-access-token)" \
  https://cloudresourcemanager.googleapis.com/v1/projects
```

It's important to know that to successfully make an API call, you need all of the following conditions:
* An access token that was granted the necessary __scopes__ for the operation. You can find the full list of scopes used by Google services [here](https://developers.google.com/identity/protocols/oauth2/scopes)
* Sufficient __IAM permissions__. GCP products usually have a page in their documentation dedicated to discussing permissions and roles involved, but there's also a useful [full reference page](https://cloud.google.com/iam/docs/permissions-reference).
* The target __API__ needs to be __enabled__ in the target project. You can use the 'APIs and Services' page on the GCP web console or the [Service Usage API](https://cloud.google.com/service-usage/docs/reference/rest) to manage services.
* __API Quotas__ also need to allow your call. This usually becomes important when you are using a service account to act on resources in other projects. By default, the quota will be counted in the project where the service account was created, and in this case the _API needs to be enabled in that project too_. You can learn more about this topic [here](https://cloud.google.com/docs/quota). In short, you can specify the quota project in [gcloud commands](https://cloud.google.com/sdk/gcloud/reference#--billing-project), or set the quota project for [ADC clients](https://cloud.google.com/sdk/gcloud/reference/auth/application-default/set-quota-project). This translates to additional [HTTP headers](https://github.com/googleapis/google-auth-library-nodejs/blob/74be65ddd4319ed5acd6ab8aaef5a7bb1e8aa69b/src/auth/authclient.ts#L138) in the end. In larger Terraform provisioning pipelines it's not conveniently possible to set this value for each resource, so it's a common approach to enable all affected APIs in the projects where the service accounts are created.
 
### Service Account Impersonation
Service Accounts can be used in various ways in various scenarios. It's a common pattern to create service accounts for specific system roles and then let end users or other service accounts _act as_ this particular service account.

A very common use case is an infrastructure provisioning pipeline with Terraform. A Terraform pipeline is usually executed on a CI/CD platform, for example on Cloud Build, but it's often also necessary to execute the pipeline locally from a developer machine. In this case, there's a dedicated service account created for this infrastructure provisioning system role, and then we let both the Cloud Build default service account and the devops engineers _act as_ this account. See the [official doc here](https://cloud.google.com/iam/docs/impersonating-service-accounts).

This scenario can also be set up by exporting a key file for the target service accounts and moving that key file to the developer workstations and the CI/CD pipeline executors, but this approach has obvious vulnerabilities.

Instead of working directly with private keys, we can rely on the [IAM Credentials API](https://cloud.google.com/iam/docs/reference/credentials/rest/v1/projects.serviceAccounts).

The most basic scenario here relies on the [`generateAccessToken` method](https://cloud.google.com/iam/docs/reference/credentials/rest/v1/projects.serviceAccounts/generateAccessToken). The callers of this API need to have the Service Account Token Creator role on the target service account.

This impersonation mechanism is not specified in the Application Default Credentials standard, it only supports an exported service account key. This means that you need to specifically include client libraries to do this kind of stuff.

This API method also supports providing a _delegation chain_. In case for example SA1 would generate a token for SA2, then SA2 would generate a token for SA3, this chained token generation can be performed in a single API call.

Terraform, which also works with the Google-provided Golang client libraries, [provides two ways](https://cloud.google.com/blog/topics/developers-practitioners/using-google-cloud-service-account-impersonation-your-terraform-code) to perform service account impersonation. The Terraform Google provider for example handles the environment variable [here](https://github.com/hashicorp/terraform-provider-google/blob/1285699df1e3d9bed5c5f53ecd1c71468ed43823/google/config.go#L1114). It's useful to know that if service account impersonation is in use, API errors from the impersonation flow itself are often logged misleadingly as if they were reported from the provider API call itself, so for example an Unauthorized error logged from a compute engine instance creation can actually come from the access token API.

The `gcloud` command-line client also provides the [--impersonate-service-account](https://cloud.google.com/sdk/gcloud/reference#--impersonate-service-account) option for all its commands.

For service account impersonation, the `iam.serviceAccounts.getAccessToken` is usually sufficient, which is included in the Service Account Token Creator role. This will also allow the caller to use the generated token to authorize further API calls. There's another important permission, the somewhat misleadingly named `iam.serviceAccounts.actAs`, which is included in the Service Account User role. This permission is required to _associate this service account to resources_, for example to create a VM with this service account. This permission is also required for example to SSH into a VM instance which has that service account configured.

The Golang client library provides the [impersonate](https://pkg.go.dev/google.golang.org/api/impersonate) package. You can also perform the impersonation with your custom code and then provide the resulting access token using [StaticTokenSource](https://pkg.go.dev/golang.org/x/oauth2#StaticTokenSource).

Node.js and Java client libraries don't seem to provide impersonation features (apart from the exported service account key based), and they also don't provide classes to work with an existing access token.

### User impersonation
It's also possible to impersonate end users, but strictly speaking this is not a use case on GCP itself. This is used for example if an application needs to access Calendar data of Google Workspace users. This scenario could also be performed by doing a full OAuth2 flow with each affected user and storing the refresh token on the application servers. Domain-wide delegation, however, allows an application to generate tokens for users without their interaction.

This is a common use case when an application is installed on a Google Workspace domain. The application needs to have an OAuth2 client id already generated, and then this client id can be granted access with specific scopes to an entire domain or specific Organizational Units. This can be done manually on the Workspace admin screens or this can also automatically happen when installing applications from the Workspace Marketplace.

Service Accounts also have an OAuth2 Client Id which can be used for such authorization purposes.

Common code examples of such scenarios (for example [here](https://developers.google.com/admin-sdk/directory/v1/guides/delegation)) work with exported service account keys, and it's rarely mentioned that you can also work with a mechanism based on [`signJwt`](https://cloud.google.com/iam/docs/reference/credentials/rest/v1/projects.serviceAccounts/signJwt). Exported service account keys are used basically for the same purpose - to sign a JWT token and then use that encoded token to generate an access token. JWT signing can be performed on Google's side with this API call.

For example:
```javascript
const accessToken = '<<existing access token from env>>';
const jwt = {
        'iss': 'sa-name@sa-project.iam.gserviceaccount.com',
        'scope': "openid https://www.googleapis.com/auth/userinfo.profile https://www.googleapis.com/auth/userinfo.email",
        'sub': 'target.user@domain.com',
        'aud': 'https://oauth2.googleapis.com/token',
        'iat': Math.floor(new Date().getTime() / 1000),
        'exp': (Math.floor(new Date().getTime() / 1000) + 60 * 60)
    }
request = {
        "delegates": [],
        "payload": JSON.stringify(jwt)
    }
const signed = await axios.post("https://iamcredentials.googleapis.com/v1/projects/-/serviceAccounts/sa-name@sa-project.iam.gserviceaccount.com:signJwt", request, { headers: {
        'Authorization': 'Bearer ' + accessToken
    }});
 // perform token exchange
const exchanged = await axios.post('https://oauth2.googleapis.com/token', {
        'grant_type': "urn:ietf:params:oauth:grant-type:jwt-bearer",
        'assertion': signed.data.signedJwt
    });
// check the exchanged token
const email = await axios.get('https://openidconnect.googleapis.com/v1/userinfo', { headers: {
        'Authorization': 'Bearer ' + exchanged.data.access_token
    }})
```

This kind of flow is unfortunately not supported in Node.js or Java client libraries.

### Workload Identity Federation
When your workload is running somewhere inside GCP, it's very likely that authorization for GCP service calls can be set up in a way that avoids managing service account keys or other stored credentials.

For execution environments outside GCP, the metadata server won't be available, thus the default solution is often an exported credential file.

[Workload Identity Federation](https://cloud.google.com/iam/docs/workload-identity-federation) provides a way to avoid using such exported credentials.

#### Shifting source of Trust
When you export a service account key file, a pair of keys is generated. The private key will be in the downloaded file and will be used for JWT token signature. GCP will verify the signature with the 'public' pair of the key.

This way, when verifying the JWT signature, Google uses its internally stored 'public' key.

The safety of this solution largely relies on that the downloaded key file is securely managed.

With Workload Identity Federation, you register an external OpenID Connect provider in Google IAM, and associate service accounts with a set of external identities. This external OIDC provider can be AWS or Azure for example, among others.

When you register such a provider in a Workload Identity Pool, you basically say that _the public keys published by these providers should be trusted as JWT token signers_.

As we discussed above, OIDC requires a provider to publish a [/.well-known/openid-configuration](https://accounts.google.com/.well-known/openid-configuration) file, which includes a field named `jwks_uri`. In case of GCP this is [https://www.googleapis.com/oauth2/v3/certs](https://www.googleapis.com/oauth2/v3/certs) - you can see certificates here which are used for JWT token signing. Other providers will publish keys similarly.

In short, instead of GCP trusting its own copy of the public key of a service account key pair, it will trust the publicly available certificate that you registered to the identity pool.

The safety of this solution relies on whether these external OIDC providers only sign JWT tokens in properly authorized cases.

#### Under the hood

To authorize a GCP API call from such an external environment, you basically follow these steps:

```javascript
    // Sign JWT somehow with the private key.
    // This is done by some API call on the external side, you don't normally directly access this key
    const jwt = await new jose.SignJWT({
        "iss": "https://my-oidc-provider.com",
        "iat": now,
        "exp": now + 3600,
        "sub": "sa-federated-actor@project-id.iam.gserviceaccount.com",
        "aud": "//iam.googleapis.com/projects/123456789/locations/global/workloadIdentityPools/oidc-demo/providers/oidc-demo-provider"
    })
        .setProtectedHeader({ alg: 'RS256', kid: 'my-key-id' })
        .sign(ecPrivateKey);

    // Use the signed JWT on the GCP Security Token Service API
    const stsToken = await axios.post("https://sts.googleapis.com/v1/token", {
        grantType: "urn:ietf:params:oauth:grant-type:token-exchange",
        audience: "//iam.googleapis.com/projects/123456789/locations/global/workloadIdentityPools/oidc-demo/providers/oidc-demo-provider",
        scope: "https://www.googleapis.com/auth/cloud-platform",
        requestedTokenType: "urn:ietf:params:oauth:token-type:access_token",
        subjectToken: jwt,
        subjectTokenType: "urn:ietf:params:oauth:token-type:jwt"
    });

    // Exchange the STS Token for a regular Access token
    const accessToken = await axios.post("https://iamcredentials.googleapis.com/v1/projects/-/serviceAccounts/sa-federated-actor@project-id.iam.gserviceaccount.com:generateAccessToken", {
        scope: ["https://www.googleapis.com/auth/compute.readonly"]
    }, { headers: {
        "Authorization": "Bearer " + stsToken.data.access_token
    }});

    // Make an example API call
    const instances = await axios.get("https://compute.googleapis.com/compute/v1/projects/project-id/zones/europe-west1-b/instances", { headers: {
        "Authorization": "Bearer " + accessToken.data.accessToken
    }})
```

The [documentation](https://cloud.google.com/iam/docs/using-workload-identity-federation) provides examples of how this is done for each external provider.

Remember that OIDC doesn't prescribe having an actual user directory or any fancy service. You can set up a simple external identity implementation for yourself by publishing a standard OIDC description file on a public HTTPS server, along with the public certificates. Then you can use the private keys to sign JWT requests as shown above and exchange them for STS tokens in the pools that you've registered with this provider.

The key benefit of using Workload Identity Federation though is that you can safely trust these existing, well-known identity providers that their private keys won't be misused, and that you can easily sign JWT tokens when your code is executed on these environments.

#### Application-default credentials
When you configure external identities to be associated with specific service accounts, you can download a json file that can be used to configure ADC.

This file will look like this:
```json
{
  "type": "external_account",
  "audience": "//iam.googleapis.com/projects/123456789/locations/global/workloadIdentityPools/oidc-demo/providers/oidc-demo-provider",
  "subject_token_type": "urn:ietf:params:oauth:token-type:jwt",
  "token_url": "https://sts.googleapis.com/v1/token",
  "credential_source": {
  },
  "service_account_impersonation_url": "https://iamcredentials.googleapis.com/v1/projects/-/serviceAccounts/sa-federated-actor@project-id.iam.gserviceaccount.com:generateAccessToken"
}
```

This file can be picked up by ADC implementation in client libraries. [This repository](https://github.com/salrashid123/gcpcompat-oidc) also gives you a detailed technical walkthrough on the process.

#### Github Actions
The integration with Github OIDC is especially notable because infrastructure provisioning pipelines often need to work with highly privileged service accounts.

The integration with Github Action also provides build steps configurable in your pipelines to perform all the low-level API calls that are needed to get an access token.

You need to keep in mind though that a Github Action pipeline declared in a repository might be misused for privilege escalation if the developers having write access to the repository don't have the pipeline's privileges anyway. Branch protection has to be used in conjunction with protected environments to fully prevent escalation. Cloud Build is still often preferred for highly privileged provisioning pipelines.

#### Kubernetes Workload Identity
As also mentioned above, Google Kubernetes Engine is a bit of a special environment, because a single VM can host multiple workloads.

By default, any pod running on the VM can access the metadata server of the VM, making them able to make API calls on behalf of the node's service account. In this setup all workloads in the node pool will have the same service identity. Moreover, this service identity needs to be the same as the service account the kubelet uses.

The Metadata Concealment option lets you prevent pods from accessing the metadata server. This is useful if the workloads don't need to make GCP API calls, but in case they need to, you would end up storing service account keys on the cluster.

By configuring Workload Identity for the cluster, you can associate GCP service account to Kubernetes service accounts, and if a pod is configured to work with a specific Kubernetes service account, it will see a metadata server endpoint configured to return access tokens for the associated GCP service account.

All the technical details we discussed above are transparently handled, client libraries running in these pods will just work with the ADC mechanism.

### Identity-aware Proxy
This is not strictly related to the topic, but it's a very important feature on GCP to know about.

Identity-Aware Proxy is basically a TCP-over-HTTPS proxy. Connections are authorized with the same mechanisms we've been discussing. After the authorization is made, roughly speaking TCP packets are sent in consequent HTTP requests.

As this architecture suggests, this is only suitable for administrative access, but it's very handy for that purpose.

Assuming you have a VM without a publicly accessible IP, you can still open an SSH connection to it if you have sufficient permissions and if TCP connections are allowed to the VM on port 22 from the predefined IAP proxy IP range.

This mechanism works if you open an SSH connection in the browser, but you can also use the gcloud command line, for example

```
gcloud compute ssh my-instance --project project --zone zone
```

There's an explicit `--tunnel-through-iap` option that you can specify to force using the IAP tunnel. This can come handy if the target instance has a public IP otherwise, but firewall rules disallow the SSH port from being accessible from external IP addresses.

This way you can also open further SSH tunnels to other nodes in your VPC, for example a Cloud SQL instance.

```
gcloud compute ssh my-instance --project project --zone zone -- -L 3306:10.1.0.3:33306
```

---
And that should be it, I hope these details help you troubleshoot problems.