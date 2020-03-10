### **Adapter and `TurnContext`**

```mermaid
    graph LR
        BotEndpoint["#quot;api/messages#quot; endpoint in controller"] -- calls --> adapter.ProcessActivity["ProcessActivity()"] -- which creates --> TurnContext 
```

On `TurnContext` Initialization
```mermaid
    graph LR
        Adapter -- stored as member in --> TurnContext
```

`OAuthPrompt` has various methods* that uses `BotAdapter` within its logic:
```mermaid
    graph LR
    OAuthPrompt -- takes --> BotAdapter -. from .->  TurnContext
```
* `OAuthPrompt` methods that use `BotAdapter`: `BeginDialogAsync()`, `GetUserTokenAsync()`, `SignUserOutAsync()`, `SendOAuthCardAsync()`, `RecognizeTokenAsync()`

### **Class Diagrams of `OAuthPrompt` and How It Acquires Tokens**
- [C#](#c-token-provider-in-oauthprompt)
- [JS](#js-token-provider-in-oauthprompt)

#### **C#: Token Provider in `OAuthPrompt`:**

`OAuthPrompt` uses a `BotFrameworkAdapter` that implements `ICredentialTokenProvider` to acquire tokens.

```mermaid
    classDiagram
        class OAuthPrompt

        class BotAdapter {
        }
        <<abstract>> BotAdapter

        class ICredentialTokenProvider {
            + GetUserTokenAsync()
            + GetOauthSignInLinkAsync()
            + SignOutUserAsync()
            + GetTokenStatusAsync()
            + GetAadTokenAsync()
        }
        <<Interface>> ICredentialTokenProvider

        OAuthPrompt o-- ICredentialTokenProvider: uses as token provider
        BotFrameworkAdapter --|> BotAdapter: derives from
        BotAdapter --o TurnContext: is a member of
        ICredentialTokenProvider <|-- BotFrameworkAdapter
        BotFrameworkAdapter --> OAuthClient : creates to get tokens
```

#### Use `AppCredentials` to create an `OAuthClient`
- The `OAuthPrompt`'s `ICredentialTokenProvider` creates an `OAuthClient` to send a request to get a token.
- You must use `ServiceClientCredentials` in order to initialize an `OAuthClient` instance.

```mermaid
    graph LR
        OAuthPrompt -- creates --> OAuthClient -. with .-> ServiceClientCredentials
        OAuthClient -- to use --> UserToken -- to get --> Token
```

#### `AppCredentials` Class Diagram

```mermaid
    classDiagram
        class ServiceClientCredentials {
        }
        <<abstract>> ServiceClientCredentials

        class AppCredentials {
        }
        <<abstract>> AppCredentials
    
        ServiceClientCredentials <|-- AppCredentials

        AppCredentials --|> MicrosoftAppCredentials
        MicrosoftAppCredentials --|> MicrosoftGovernmentAppCredentials
        AppCredentials --|> CertificateAppCredentials
```
* `ServiceClientCredentials` is an [MS REST class](https://docs.microsoft.com/en-us/dotnet/api/microsoft.rest.serviceclientcredentials?view=azure-dotnet).

#### `BotFrameworkAdapter` Creates `OAuthClient`
``` mermaid
    graph LR
        BotFrameworkAdapter -. with .-> AppCredsFromSetting[AppCredentials from OAuthPromptSettings]
        BotFrameworkAdapter -. or with .-> AppCredsFromCache[AppCredentials from Adapter's credential cache]
        BotFrameworkAdapter -. or with .-> builtAppCredentials[newly built AppCredentials]
        
        AppCredsFromSetting -->  OAuthClient[creates OAuthClient]
        AppCredsFromCache -->  OAuthClient
        builtAppCredentials -->  OAuthClient

```

#### `OAuthClient` Class Diagram

```mermaid
    classDiagram
        class OAuthClient

        class ServiceClient {
        }
        <<abstract>> ServiceClient

        class IOAuthClient {
            + BaseUri
            + SerializationSettings
            + DeserializationSettings
            + Credentials
            + BotSignIn
            + UserToken
        }

        OAuthClient --|> ServiceClient: derives from
        OAuthClient --|> IOAuthClient: implements
        IOAuthClient --|> IDisposable

        class IUserToken {
            + GetTokenWithHttpMessagesAsync()
            + GetAadTokensWithHttpMessagesAsync()
            + SignOutWithHttpMessagesAsync()
            + GetTokenStatusWithHttpMessagesAsync()
        }

        OAuthClient *-- IUserToken: uses to get token

        class UserToken {
            + GetTokenWithHttpMessagesAsync()
        }

        IUserToken <|-- UserToken: gets Token from UserToken response
```
* `ServiceClient` is an MS REST class.

#### **JS: Token Provider in `OAuthPrompt`:**

`OAuthPrompt` uses a `BotFrameworkAdapter` that implements `ExtendedUserTokenProvider` to acquire tokens.

```mermaid
    classDiagram
        class OAuthPrompt

        class BotAdapter {
        }
        <<abstract>> BotAdapter

        class ExtendedUserTokenProvider {
            + getUserToken()
            + signOutUser()
            + getSignInLink()
            + getAadTokens()
            + getSignInResource()
            + exchangeToken()
        }

        OAuthPrompt o-- ExtendedUserTokenProvider: uses as token provider
        ExtendedUserTokenProvider <|-- BotFrameworkAdapter
        BotFrameworkAdapter --> TokenApiClient : creates to get tokens
        BotFrameworkAdapter --|> BotAdapter: derives from
        BotAdapter --o TurnContext: is a member of
```

#### `TokenApiClient` Class Diagram
```mermaid
    classDiagram
        class TokenApiClient {
            + botSignIn
            + userToken
        }

        class TokenApiClientContext {
            + baseUri
            + requestContentType
            + credentials
        }

        TokenApiClient --|> TokenApiClientContext: extends
        TokenApiClientContext --|> ServiceClient: extends

        class UserToken {
            + getToken()
        }

        TokenApiClient *-- UserToken: uses to get token
```

* `TokenApiClient` and `TokenApiClientContext` are classes generated by auto-rest.
* `ServiceClient` is an [msrest class](https://github.com/Azure/ms-rest-js/blob/master/lib/serviceClient.ts).