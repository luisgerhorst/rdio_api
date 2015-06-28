# Rdio API

This is an Erlang wrapper for the [Rdio API](http://www.rdio.com/developers/). It automatically limits the number of requests according to the API's rate limit and refreshes the OAuth 2.0 access tokens when they expire.

## Config

Set your applications Client ID and Client Secret as environment variables in string format before you start the application.

```erl
application:set_env(rdio_api, client_secret, "my-client-secret-here"),
application:set_env(rdio_api, client_id, "my-client-id-here")
```

Optionally you may specify the rate limit for your app (default is 10 calls per second, `{10, 1000}`).

```erl
application:set_env(rdio_api, rate_limit, {MaxCalls, TimeframeInMilliseconds})
```

## Start

Start the application before making requests:

```erl
application:ensure_all_started(rdio_api)
```

## Usage

### Authorization

Using the `rdio_api_authorization:tokens_with_AuthorizationMethod` or `rdio_api_authorization:tokens/3` functions you can obtain the opaque type `tokens()`. You pass the type into every API request and get potentially refreshed tokens back.

#### Types

`rdio_api_authorization` exports the following types:

```erl
-opaque tokens().

-type refresh_token() :: string().
-type access_token() :: string().

-type client_id() :: string().
-type client_secret() :: string().
```

In addition to that the following types apply in the documentation:

```erl
TokenEndpointResponse = {ok, tokens()} | {error, {400, {Error :: binary(), ErrorDesciption :: binary()}} | {HttpCode, HttpBody}}
```

#### Accessors

Manually create tokens or access the internals of the opaque type:

```erl
rdio_api_authorization:refresh_token(Tokens) -> string()
rdio_api_authorization:access_token(Tokens) -> string()
rdio_api_authorization:expires(Tokens) -> non_neg_integer()
rdio_api_authorization:tokens(RefreshToken :: string(), AccessToken :: string(), ExpirationTimestamp :: non_neg_integer()) -> tokens()
```

```erl
rdio_api_authorization:client_id() -> string()
rdio_api_authorization:client_secret() -> string()
```

#### Authorization Code Grant

Obtain the URL you have to redirect the user to. You can also obtain the Client ID usign `rdio_api_authorization:client_id() -> string()` and construct the URL manually.

```erl
rdio_api_authorization:code_authorization_url(RedirectUri :: string()) -> string()
rdio_api_authorization:code_authorization_url(RedirectUri :: string(), Scope :: string()) -> string()
rdio_api_authorization:code_authorization_url(RedirectUri :: string(), Scope :: string(), State :: string()) -> string()
```

When the user has allowed your application to access their account and you have received the authorization code, turn it into the opque tokens type:

```erl
rdio_api_authorization:tokens_with_authorization_code(Code :: string(), RedirectUri :: string()) -> TokenEndpointResponse
```

#### Implicit Grant

Get the URL you have to redirect the user to:

```erl
rdio_api_authorization:token_authorization_url(RedirectUri :: string()) -> string()
rdio_api_authorization:token_authorization_url(RedirectUri :: string(), Scope :: string()) -> string()
rdio_api_authorization:token_authorization_url(RedirectUri :: string(), Scope :: string(), State :: string()) -> string()
```

#### Resource Owner Credential

After polling the user for his username and password, you can retreive the tokens:

```erl
tokens_with_resource_owner_credentials(Username :: string(), Password :: string()) -> TokenEndpointResponse
tokens_with_resource_owner_credentials(Username :: string(), Password :: string(), Scope :: string()) -> TokenEndpointResponse
```

#### Client Credentials

```erl
tokens_with_client_credentials() -> TokenEndpointResponse
tokens_with_client_credentials(Scope :: string()) -> TokenEndpointResponse
```

#### Device Code Grant

```erl
start_device_code_grant() -> Return
start_device_code_grant(Scope) -> Return
```

Types:

```erl
Return = {ok, DeviceCode :: binary(), VerificationUrl :: binary(), ExpirationTimestamp, PollingInterval} | {error, {HttpCode, HttpBody}}
```

Note that `ExpirationTimestamp` is not the number of seconds until the device code expires, but instead the Unix time (in seconds) when it is expected to expire.

You can then poll the token endpoint with:

```erl
tokens_with_device_code(DeviceCode :: binary() | string()) -> TokenEndpointResponse
```

### Requests

A simple API request:

```erl
rdio_api:request(Method :: string(), Arguments :: [{Key :: string(), Value :: string()}], Tokens :: tokens()) -> {ok, MethodResult :: map(), NewTokens :: tokens()} | {error, #{ErrorType => ErrorReason} | #{tokens => NewTokens, ErrorType => ErrorReason}}
```

`MethodResult` is the parsed response from Rdio (using [`jiffy:decode/2`](https://github.com/davisp/jiffy#jiffydecode12)).

Sometimes you may want to perform multiple requests in quick succession, for that you can use:

```erl
rdio_api:run(fun (Request) ->
    {ok, _MethodResult, Tokens2} = Request(Method1, Args1, Tokens1),
    {ok, _MethodResult, Tokens3} = Request(Method2, Args2, Tokens2),
    ...
end)
```

`Request` behaves like `rdio_api:request/3`, but here the (forced) delay between the requests is _always_ the timeframe specified in the applications rate limit, e.g. one second for regular applications. Note that this method may, under low load, actually be slower then calling `rdio_api:request/3` multiple times. Anyway, if you want a guarantee, use this function.

## Examples

### Complete setup guide

Create an app at [www.rdio.com/developers/your-apps](http://www.rdio.com/developers/your-apps/). Add `http://localhost/` as redirect URI.

```
$ cd /path/to/rdio_api
$ rebar get-deps
$ rebar compile
$ rebar shell
1> application:set_env(rdio_api, client_id, "MyAppsClientIDHere").
2> application:set_env(rdio_api, client_secret, "MyAppsClientSecretHere").
3> application:ensure_all_started(rdio_api).
4> RedirectUri = "http://localhost/".
5> rdio_api_authorization:authorization_url(RedirectUri).
```

Open the shown URL in your browser and allow your app to access your account. You will be redirected to a URL that looks like this `http://localhost/?code=Code`, copy the `Code` part to your clipboard and proceed.

```
6> Code = "MyCopiedCodeHere".
7> {ok, Tokens} = rdio_api_authorization:tokens_with_authorization_code(Code, RedirectUri).
8> {ok, #{<<"firstName">> := UserFirstNameBinary, <<"lastName">> := UserLastNameBinary}, Tokens2} = rdio_api:request("currentUser", [], Tokens).
```
