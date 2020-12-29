# Create and authorize a Ruqqus Application
The OAuth2 authorization flow is used to enable users to authorize third-party applications to access their Ruqqus account without having to provide their login information to the application.

This page explains how to obtain API application keys, how to prompt a user for authorization, and how to obtain and use access tokens.

## Step 1: Create your Application
In the apps tab of Ruqqus settings, fill in and submit the form to request new API keys. You will need:

- an application name
- a Redirect URI, or a comma-separated list of redirect URIs. May not use HTTP unless using localhost (use HTTPS instead).
- a brief description of what your application is intended to do

Don't worry too much about accuracy; you will be able to change all of these later.

Ruqqus administrators will review and approve or deny your request for API keys. You'll know when your request has been approved when a client ID and secret appear in your application information.

DO NOT reveal your Client Secret. Anyone with your Client Secret will be able to pretend to be you. You are responsible for keeping your Client Secret a secret!

![](../assets/images/reader.png)


## Step 2: Prompt Your User for Authorization
Send your user to https://ruqqus.com/oauth/authorize, with the following URL parameters:

- `client_id` - Your application's Client ID
- `redirect_uri` - The redirect URI (or one of the URIs) specified in your application information. Must not use HTTP unless using localhost (use HTTPS instead).
- `state` - This is your own anti-cross-site-forgery token. We don't do anything with this, except give it to the user to pass back to you later. You are responsible for generating and validating the state token.
- `scope` - A comma-separated list of permission scopes that you are requesting. Valid scopes are: `identity`, `create`, `read`, `update`, `delete`, `vote`, and `guildmaster`.
- `permanent` - optional. Set to true if you are requesting indefinite access. Omit if not.

If done correctly, the user will see that your application wants to access their Ruqqus account, and be prompted to approve or deny the request.

![](../assets/images/reader.png)



## Step 3: Catch the redirect
The user clicks "Authorize". Ruqqus will redirect the user's browser to GET the designated redirect URI. The following URL parameters will be included, which your server should process:

- `code` - a single-use authorization code.
- `state` - The state token from earlier.

Validate the state token. How you do this is up to you.

## Step 4: Exchange code for access token
Make a POST request to https://ruqqus.com/oauth/grant. Include the following form parameters:

- `client_id` - Your application's Client ID
- `client_secret` - Your application's Client Secret
- `grant_type` - Set to the word "code"
- `code` - The code parameter that was given to you in the previous step.

Python example:

    import requests
    import pprint

    code=request.args.get("code")

    headers={"User-Agent": "Porpl Reader v1 by @captainmeta4"}
    url="https://ruqqus.com/oauth/grant"
    data={"client_id": my_client_id,
          "client_secret": my_client_secret,
          "grant_type": "code",
          "code": code
          }

    r=requests.post(url, headers=headers, data=data)

    pprint.pprint(r.json())
If everything is good, we will respond with the following (example) JSON body:

    {
    "access_token":         #Access token
    "scopes":               #Comma-separated list of scopes included in authorization
    "expires_at":           #Unix epoch integer time at which access token expires
    "token_type": "Bearer"
    "refresh_token":        #This key is omitted in temporary authorizations
    }
Store the access and refresh tokens. You should also store expiration timestamp and the scopes list, so that you pre-emptively avoid sending requests to Ruqqus that won't be accepted.

## Step 5: Using the Access Token
To use the access token, include the following header in subsequent API requests to Ruqqus: Authorization: Bearer access_token_goes_here

Python example, presuming that the application has obtained a valid read authorization:

    import requests
    import pprint

    headers={"Authorization": "Bearer " + access_token,
             "User-Agent": "Porpl Reader v1 by @captainmeta4"}
    url="https://ruqqus.com/api/v1/front/listing"

    r=requests.get(url, headers=headers)

    pprint.pprint(r.json())
The expected result of this would be a large JSON representation of the submissions that make up the user's personal front page.

## Step 6: Maintain Access
Access tokens expire one hour after they are issued. To maintain ongoing access, you will need to use the refresh token to obtain a new access token.

Make a POST request to https://ruqqus.com/oauth/grant with the following form parameters:

- `client_id` - Your application's Client ID
- `client_secret` - Your application's Client Secret
- `grant_type` - set to the word "refresh"
- `refresh_token` - the refresh token which you obtained during Step 3

Python example:

    import requests
    import pprint

    headers={"User-Agent": "Porpl Reader v1 by @captainmeta4"}
    url="https://ruqqus.com/oauth/grant"
    data={"client_id": my_client_id,
          "client_secret": my_client_secret,
          "grant_type": "refresh",
          "refresh_token": my_users_refresh_token
          }

    r=requests.post(url, headers=headers, data=data)

    pprint.pprint(r.json())
A successful refresh will result in the following response:

    {
    "access_token":         #Access token
    "expires_at":           #Unix epoch integer time at which access token expires
    }
You should overwrite the previous access code and expiration timestamp with this new information. Continue to use the access token as in Step 5, until it expires.