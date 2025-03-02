<!-- markdownlint-disable MD002 MD041 -->

## Configure Your Application to Use Auth0

[Universal Login](/hosted-pages/login) is the easiest way to set up authentication in your application. We recommend using it for the best experience, best security and the fullest array of features. This guide will use it to provide a way for your users to log in to your ASP.NET MVC 5 application.

::: note
You can also create a custom login for prompting the user for their username and password. To learn how to do this in your application, follow the [Custom Login sample](https://github.com/auth0-samples/auth0-aspnet-owin-mvc-samples/tree/master/Samples/custom-login).
:::

### Install and configure the OpenID Connect middleware

::: note
  This quickstart makes use of OWIN middleware and as such, you need to use OWIN in your application. If your application is not currently making use of OWIN, please refer to Microsoft's <a href="https://docs.microsoft.com/en-us/aspnet/aspnet/overview/owin-and-katana/">OWIN documentation</a> to enable it in your application.
:::

The easiest way to enable authentication with Auth0 in your ASP.NET MVC application is to use the OWIN OpenID Connect middleware which is available in the `Microsoft.Owin.Security.OpenIdConnect` NuGet package, so install that first:

```bash
Install-Package Microsoft.Owin.Security.OpenIdConnect
```

You must also install the following middleware library to enable cookie authentication in your project:

```bash
Install-Package Microsoft.Owin.Security.Cookies
```

:::note
There are issues when configuring the OWIN cookie middleware and System.Web cookies at the same time. Please read about the [System.Web cookie integration issues doc](https://github.com/aspnet/AspNetKatana/wiki/System.Web-response-cookie-integration-issues) to learn about how to mitigate these problems
:::

Now go to the `Configuration` method of your `Startup` class and configure the cookie middleware as well as the Auth0 middleware.

```cs
// Startup.cs
using Microsoft.IdentityModel.Protocols.OpenIdConnect;
using Microsoft.IdentityModel.Tokens;
using Microsoft.Owin;
using Microsoft.Owin.Host.SystemWeb;
using Microsoft.Owin.Security;
using Microsoft.Owin.Security.Cookies;
using Microsoft.Owin.Security.OpenIdConnect;
using MvcApplication.Support;
using Owin;

public void Configuration(IAppBuilder app)
{
    // Configure Auth0 parameters
    string auth0Domain = ConfigurationManager.AppSettings["auth0:Domain"];
    string auth0ClientId = ConfigurationManager.AppSettings["auth0:ClientId"];
    string auth0RedirectUri = ConfigurationManager.AppSettings["auth0:RedirectUri"];
    string auth0PostLogoutRedirectUri = ConfigurationManager.AppSettings["auth0:PostLogoutRedirectUri"];

    // Set Cookies as default authentication type
    app.SetDefaultSignInAsAuthenticationType(CookieAuthenticationDefaults.AuthenticationType);
    app.UseCookieAuthentication(new CookieAuthenticationOptions
    {
        AuthenticationType = CookieAuthenticationDefaults.AuthenticationType,
        LoginPath = new PathString("/Account/Login"),

        // Configure SameSite as needed for your app. Lax works well for most scenarios here but
        // you may want to set SameSiteMode.None for HTTPS
        CookieSameSite = SameSiteMode.Lax,

        // More information on why the CookieManager needs to be set can be found here: 
        // https://github.com/aspnet/AspNetKatana/wiki/System.Web-response-cookie-integration-issues
        CookieManager = new SameSiteCookieManager(new SystemWebCookieManager())
    });

    // Configure Auth0 authentication
    app.UseOpenIdConnectAuthentication(new OpenIdConnectAuthenticationOptions
    {
        AuthenticationType = "Auth0",

        Authority = $"https://{auth0Domain}",

        ClientId = auth0ClientId,

        RedirectUri = auth0RedirectUri,
        PostLogoutRedirectUri = auth0PostLogoutRedirectUri,

        TokenValidationParameters = new TokenValidationParameters
        {
            NameClaimType = "name"
        },

        // More information on why the CookieManager needs to be set can be found here: 
        // https://docs.microsoft.com/en-us/aspnet/samesite/owin-samesite
        CookieManager = new SameSiteCookieManager(new SystemWebCookieManager()),

        Notifications = new OpenIdConnectAuthenticationNotifications
        {
            RedirectToIdentityProvider = notification =>
            {
                if (notification.ProtocolMessage.RequestType == OpenIdConnectRequestType.Logout)
                {
                    var logoutUri = $"https://{auth0Domain}/v2/logout?client_id={auth0ClientId}";

                    var postLogoutUri = notification.ProtocolMessage.PostLogoutRedirectUri;
                    if (!string.IsNullOrEmpty(postLogoutUri))
                    {
                        if (postLogoutUri.StartsWith("/"))
                        {
                            // transform to absolute
                            var request = notification.Request;
                            postLogoutUri = request.Scheme + "://" + request.Host + request.PathBase + postLogoutUri;
                        }
                        logoutUri += $"&returnTo={ Uri.EscapeDataString(postLogoutUri)}";
                    }

                    notification.Response.Redirect(logoutUri);
                    notification.HandleResponse();
                }
                return Task.FromResult(0);
            }
        }
    });
}
```

It is essential that you register both the cookie middleware and the OpenID Connect middleware, as they are required (in that order) for the authentication to work. The OpenID Connect middleware will handle the authentication with Auth0. Once the user has authenticated, their identity will be stored in the cookie middleware.

In the code snippet above, note that the `AuthenticationType` is set to **Auth0**. This will be used in the next section to challenge the OpenID Connect middleware and start the authentication flow. Also note code in the `RedirectToIdentityProvider` notification event which constructs the correct [logout URL](/logout).

## Trigger Authentication

### Add Login and Logout methods

Next, you will need to add `Login` and `Logout` actions to the `AccountController`.

The `Login` action will challenge the OpenID Connect middleware to start the authentication flow. For the `Logout` action, you will need to sign the user out of the cookie middleware (which will clear the local application session), as well as the OpenID Connect middleware. For more information, you can refer to the Auth0 [Logout](/logout) documentation.

```cs
// Controllers/AccountController.cs

public class AccountController : Controller
{
    public ActionResult Login(string returnUrl)
    {
        HttpContext.GetOwinContext().Authentication.Challenge(new AuthenticationProperties
            {
                RedirectUri = returnUrl ?? Url.Action("Index", "Home")
            },
            "Auth0");
        return new HttpUnauthorizedResult();
    }

    [Authorize]
    public void Logout()
    {
        HttpContext.GetOwinContext().Authentication.SignOut(CookieAuthenticationDefaults.AuthenticationType);
        HttpContext.GetOwinContext().Authentication.SignOut("Auth0");
    }

    [Authorize]
    public ActionResult Claims()
    {
        return View();
    }
}
```

### Add Login and Logout links

To add the Login and Logout links to the navigation bar, head over to `/Views/Shared/_Layout.cshtml` and add code to the navigation bar section which displays a Logout link when the user is authenticated, otherwise a Login link. These will link to the `Logout` and `Login` actions of the `AccountController` respectively:

```html
<!-- Views/Shared/_Layout.cshtml -->

<div class="navbar navbar-inverse navbar-fixed-top">
    <div class="container">
        <div class="navbar-header">
            <button type="button" class="navbar-toggle" data-toggle="collapse" data-target=".navbar-collapse">
                <span class="icon-bar"></span>
                <span class="icon-bar"></span>
                <span class="icon-bar"></span>
            </button>
            @Html.ActionLink("Application name", "Index", "Home", new { area = "" }, new { @class = "navbar-brand" })
        </div>
        <div class="navbar-collapse collapse">
            <ul class="nav navbar-nav">
                <li>@Html.ActionLink("Home", "Index", "Home")</li>
            </ul>
            <ul class="nav navbar-nav navbar-right">
                @if (User.Identity.IsAuthenticated)
                {
                    <li>@Html.ActionLink("Logout", "Logout", "Account")</li>
                }
                else
                {
                    <li>@Html.ActionLink("Login", "Login", "Account")</li>
                }
            </ul>
        </div>
    </div>
</div>
```

### Obtain an Access Token for calling an API

If you want to call an API from your MVC application, you need to obtain an Access Token issued for the API you want to call. To receive and Access Token, pass an additional audience parameter containing the API identifier to the Auth0 authorization endpoint.

You will also need to configure the OpenID Connect middleware to add the ID Token and Access Token as claims on the `ClaimsIdentity`.

Update the OpenID Connect middleware registration in your `Startup` class as follows:

1. Set the `ResponseType` to `OpenIdConnectResponseType.Code`. This will inform the OpenID Connect middleware to extract the Access Token and store it in the `ProtocolMessage`.
1. Set `RedeemCode` to `true`.
1. Set the `ClientSecret` to the application's Client Secret, which you can find in your Auth0 dashboard.
1. Handle the `RedirectToIdentityProvider` to check to an authentication request and add the `audience` parameter.
1. Handle the `SecurityTokenValidated` to extract the ID Token and Access Token from the `ProtocolMessage` and store them as claims.

```csharp
// Startup.cs

public void Configuration(IAppBuilder app)
{
    // Some code omitted for brevity...

    string auth0ClientSecret = ConfigurationManager.AppSettings["auth0:ClientSecret"];
    string auth0Audience = ConfigurationManager.AppSettings["auth0:Audience"];

    // Configure Auth0 authentication
    app.UseOpenIdConnectAuthentication(new OpenIdConnectAuthenticationOptions
    {
        //...

        ClientSecret = auth0ClientSecret,
        ResponseType = OpenIdConnectResponseType.Code,
        RedeemCode = true,
        //...

        Notifications = new OpenIdConnectAuthenticationNotifications
        {
            SecurityTokenValidated = notification =>
            {
                notification.AuthenticationTicket.Identity.AddClaim(new Claim("id_token", notification.ProtocolMessage.IdToken));
                notification.AuthenticationTicket.Identity.AddClaim(new Claim("access_token", notification.ProtocolMessage.AccessToken));

                return Task.FromResult(0);
            },
            RedirectToIdentityProvider = notification =>
            {
                if (notification.ProtocolMessage.RequestType == OpenIdConnectRequestType.Authentication)
                {
                    // The context's ProtocolMessage can be used to pass along additional query parameters
                    // to Auth0's /authorize endpoint.
                    // 
                    // Set the audience query parameter to the API identifier to ensure the returned Access Tokens can be used
                    // to call protected endpoints on the corresponding API.
                    notification.ProtocolMessage.SetParameter("audience", auth0Audience);
                }
                else if (notification.ProtocolMessage.RequestType == OpenIdConnectRequestType.Logout)
                {
                    //...
                }
                return Task.FromResult(0);
            }
        }
    });

}
```

As the above snippet is reading the `Auth0:ClientSecret` and `Auth0:Audience` from the appSettings, ensure they exist in your web.config and has their values set to the corresponding API Identifier for which you want to be retrieving an Access Token as well as the Client Secret for the application whose Client ID has been registered.

``` xml
<configuration>
  <appSettings>
    <add key="auth0:ClientSecret" value="{CLIENT_SECRET}" />
    <add key="auth0:Audience" value="{API_IDENTIFIER}" />
  </appSettings>
</configuration>
```

To access the ID Token and Access Token from one of your controllers, cast the `User.Identity` property to a `ClaimsIdentity`, and then find the particular claim by calling the `FindFirst` method.

``` csharp
// Controllers/AccountController.cs

[Authorize]
public ActionResult Tokens()
{
    var claimsIdentity = User.Identity as ClaimsIdentity;

    // Extract tokens
    string accessToken = claimsIdentity?.FindFirst(c => c.Type == "access_token")?.Value;
    string idToken = claimsIdentity?.FindFirst(c => c.Type == "id_token")?.Value;

    // Now you can use the tokens as appropriate...
}
```