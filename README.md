# i18n
## Smart internationalization for .NET web apps
```
    PM> Install-Package I18N
```
### Introduction

The i18n library is designed to replace the use of .NET resources in favor 
of an easier, globally recognized standard for localizing web applications. 
Using this library simplifies localization by making it a first class citizen 
of views, controllers, and validation attributes.

### Features
- Globally recognized interface; localize like the big kids
- Localizes everything; views, controllers, validation attributes, and even routes!
- SEO-friendly; language selection varies the URL, and `Content-Language` is set appropriately
- Automatic; no routing changes required, just use an alias method where you want localization
- Smart; knows when to hold them, fold them, walk away, or run, based on i18n best practices

### Project Configuration

The i18n library works by modifying your HTTP traffic to perform string replacement and
patching of URLs with language tags (URL Localization). The work is done by an
HttpModule called i18n.LocalizingModule which should be enabled in your web.config file as follows:

```xml
  <system.web>
    <httpModules>
      <add name="i18n.LocalizingModule" type="i18n.LocalizingModule, i18n" />
    </httpModules>
  </system.web>
  <system.webServer> <!-- IIS7 'Integrated Mode'-specific config stuff -->
    <modules>
      <add name="i18n.LocalizingModule" type="i18n.LocalizingModule, i18n" />
    </modules>
  </system.webServer>
```

Note: The ```<system.webServer>``` element is added for completeness and may not be required.

The following ```<appSettings>``` are then required to specify the type and location 
of your application's source files:

```xml
  <appSettings>
    <add key="i18n.DirectoriesToScan" value=".." /> <!-- Rel to web.config file -->
    <add key="i18n.WhiteList" value="*.cs;*.cshtml;*.sitemap" />
  </appSettings>
```

Finally, certain behaviours of i18n may be altered at runtime on application startup. The following
code shows the most common options:

```csharp
    public class MvcApplication : System.Web.HttpApplication
    {
        protected void Application_Start()
        {
            // Change from the default of 'en'.
            i18n.LocalizedApplication.Current.DefaultLanguage = "fr";

            // Change from the of temporary redirects during URL localization
            i18n.LocalizedApplication.Current.PermanentRedirects = true;

            // This line can be used to disable URL Localization.
            //i18n.LocalizedApplication.Current.EarlyUrlLocalizerService = null;

            // Change the URL localization scheme from Scheme1.
            i18n.UrlLocalizer.UrlLocalizationScheme = i18n.UrlLocalizationScheme.Scheme2;

            // Blacklist certain URLs from being 'localized'.
            i18n.UrlLocalizer.IncomingUrlFilters += delegate(Uri url) {
                if (url.LocalPath.EndsWith("sitemap.xml", StringComparison.OrdinalIgnoreCase)) {
                    return false; }
                return true;
            };
        }
    }
```

### Usage
To localize text in your application, surround your strings with [[[ and ]]] markup
characters to mark them as translatable. That's it. 
Here's an example of localizing text in a Razor view:

```html
    <div id="content">
        <h2>[[[Welcome to my web app!]]]</h2>
        <h3><span>[[[Amazing slogan here]]]</span></h3>
        <p>[[[Ad copy that would make Hiten Shah fall off his chair!]]]</p>
        <span class="button" title="[[[Click to see plans and pricing]]]">
            <a href="@Url.Action("Plans", "Home", new { area = "" })">
                <strong>[[[SEE PLANS & PRICING]]]</strong>
                <span>[[[Free unicorn with all plans!]]]</span>
            </a>
        </span>
    </div>
```

And here's an example in a controller:

```csharp
    using i18n;
    
    namespace MyApplication
    {
        public class HomeController : LocalizingController
        {
            public ActionResult Index()
            {
                ViewBag.Message = "[[[Welcome to ASP.NET MVC!]]]";

                return View();
            }
        }
    }
```

For use in data annotations:

```csharp
    public class PasswordResetViewModel
    {
        [Required(ErrorMessage="[[[Please fill in this field]]]")]
        [Email(ErrorMessage = "[[[Email not yet correct]]]")]
        [Display(Name = "[[[Email Address]]]")]
        public string Email
        {
            get;
            set;
        }
    }
```

For use in MVC URL-Helpers or other functions that require a plain string:

```html
@Html.LabelFor(m => m.Name, "[[[First Name]]]")
```

And the same can be used for Javascript:

```html
    <script type="text/javascript">
        $(function () {
            alert("[[[Hello world!]]]");
        });
    </script>
```

### Nuggets

Strings you want to be translatable are known as messages. These in turn are 'marked-up' in your
source code as 'Nuggets'. The nugget markup allows i18n to filter the HTTP response looking for the
message strings which are replaced with translated strings where available.

A simple nugget looks like this:

```
[[[translate me]]]
```

This defines a message with the key "translate me".

Nugget markup supports formated messages as follows:

```
string.Format("[[[welcome %0, today is %1|||{0}|||{1}]]]", name, day)
```
Where the %0 and %1 tokens are replaced by the strings that replaces the {0} and {1} items, respectively.

Nugget markup also supports comments (*extracted comments* in PO parlance) to be passed to the translator like so:

```
[[[translate me///this is an extracted comment]]]
```

#### Nugget markup customization

The character sequences for marking-up nuggets ([[[, ]]], ||| and ///) were chosen on the basis that they were unlikely to clash with
common character sequences in HTML markup while at the same time being convenient for the programmer
to enter (on most keyboards).

However, recognizing that a clash remains possible and nuggets thereby being falsely detected
in source code or the HTML response, i18n supports user-defined sequences for the markup. You can
configure these in web.config as shown in the following example:

```xml
  <appSettings>
    ...
    <add key="i18n.NuggetBeginToken" value="[&[" />
    <add key="i18n.NuggetEndToken" value="]&]" />
    <add key="i18n.NuggetDelimiterToken" value="||||" />
    <add key="i18n.NuggetCommentToken" value="////" />
    ...
  </appSettings>
```

### Building PO databases

TODO: rewrite

To set up automatic PO database building, add the following post-build task to your project, after
adding `i18n.PostBuild.exe` as a project reference:

```
    "$(TargetDir)i18n.PostBuild.exe" "$(ProjectDir)"
```
    
After a successful build, this task will rip through your source code, finding everywhere you've used the `ILocalizing._("text")` alias, 
and uses this to build a master .PO template file located at `/locale/messages.pot` relative to your web application folder. After the
new template is constructed, any locales that exist inside the `/locale` folder are automatically merged with the template, so that
new strings can be flagged for further translation.

From here, you use any of the widely available PO editing tools (like [POEdit](http://www.poedit.net))
to provide locale-specific text and place them in your `/locale` folder relative to the provided language, i.e. `locale/fr`. 
If you change a PO file on the fly, i18n will update accordingly; you do _not_ need to redeploy your application.

### Route Localization

TODO: rewrite

To participate in the automatic routing features of this library, call `i18n.RouteLocalization.Enable()` 
in your startup code; this will register a global filter and route decorator to provide the feature.

I18N comes with the ability to build on top of your existing routes to automatically redirect language choice to
an appropriate URL suffix.

First of all, the set of application languages is established for which one or more translated messages exist.
Then, for each request, one of these languages is selected as the Principal Application Language (PAL) for the request.

The PAL is determined by the first of the following conditions that is met:

1. The path component of the URL is prefixed with a language tag that matches *exactly* one of the application languages. E.g. "example.com/fr/account/signup".

2. The path component of the URL is prefixed with a language tag that matches *loosely* one of the application languages.

3. The request contains a cookie called "i18n.langtag" with a language tag that matches (exactly or loosely) one of the application languages.

4. The request contains an Accept-Language header with a language that matches (exactly or loosely) one of the application languages.

5. The default application language is selected.

Where a *loose* match is made above, the URL is updated with the matched application language tag
and a redirect is issued. E.g. "example.com/fr-CA/account/signup" -> "example.com/fr/account/signup".
By default this is a temporary 302 redirect, but you can choose for it to be a permanent 301 one.

### Language Matching

TODO: update

Language matching is performed when a list of one or more user-preferred languages is matched against
a list of one or more application laguages, the goal being to choose the application languages
which the user is most likely to understand. The algorithm for this is multi-facted and multi-pass and takes the Language, 
Script and Region subtags into account.

Matching is performed once per-request to determine the principal language, and also once per GetText call. 
The multi-pass approach ensures a thorough attempt is made at matching a user's list of preferred 
languages (from their Accept-Language HTTP header). E.g. in the context of the following request:

```
User Languages: fr-CH, fr-CA  
Application Languages: fr-CA, fr, en
```

*fr-CA* will be matched first, and if no resource exists for that language, *fr* is tried, and failing
that, the default language *en* is fallen back on.

In recognition of the potential bottleneck of the GetText call (which typically is called many times per-request),
the matching algorithm is efficient for managed code (lock-free and essentially heap-allocation free).

To enable Enhanced mode in your project (the default mode being the original, Basic mode),
include the following in your Application_Start() method:

```csharp
    i18n.RouteLocalization.Enable();
```

Note that the following Chinese languages tags are normalized: zh-CN to zh-Hans, and zh-TW to zh-Hant.
It is still safe to use zh-CN and zh-TW, but internally they will be treated as equivalent to their new forms.

### A reminder about folders in a web application

Your `locale` folder is exposed to HTTP requests as-is, just like a typical log directory, so remember to block all requests
to this folder by adding a `Web.config` file. 

```xml
    <?xml version="1.0"?>
    <configuration>    
        <system.web>
            <httpHandlers>
                <add path="*" verb="*" type="System.Web.HttpNotFoundHandler"/>
            </httpHandlers>
        </system.web>
        <system.webServer>
            <handlers>
                <remove name="BlockViewHandler"/>
                <add name="BlockViewHandler" path="*" verb="*" preCondition="integratedMode" type="System.Web.HttpNotFoundHandler"/>
            </handlers>
        </system.webServer>
    </configuration>
```

### Contributing
There's lot of room for further enhancements and features to this library, and you are encouraged to fork it and
contribute back anything new. Specifically, these would be great places to add more functionality:

* Help me fix the bugs! Chances are I don't ship in your language. Fix what hurts. Please?
* Better parsing and handling of PO files for more general purposes / outside editors
