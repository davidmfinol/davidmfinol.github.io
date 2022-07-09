# Deep Linking Unity Desktop Games with Firebase Dynamic Links

Links to mobile apps with [Mobile Deep Linking](https://en.wikipedia.org/wiki/Mobile_deep_linking) is often seen as different than links intended for desktop use.
In my eyes, however, this distinction is a bit silly, as the ultimate goal is to get a user to content within an app.
There's no reason that you wouldn't want to direct users to content within an app on Windows or Mac.

Thankfully, both macOS and Windows have been adding support for deep linking through custom [URI schemes](https://en.wikipedia.org/wiki/Uniform_Resource_Identifier#Syntax), and deep linking to content has therefore become pretty ubiquitous for platforms like Discord and Spotify.
Discord and Spotify, however, are backed by massive software engineering teams that can spend the time and effort to roll out a deep linking solution that works across all supported devices.
How can a solo indie game dev such as myself hope to achieve a similar deep linking solution?

Well, my game engine of choice is Unity, and Unity has recently started adding [native support for deep links](https://docs.unity3d.com/2021.3/Documentation/Manual/deep-linking.html), so that's a good step 1.
However, there's a problem with using just the native solution: If a user clicks a deep link when they don't have the app installed, they won't go anywhere.

Most deep linking solutions get around this problem by introducing a URL that acts as a wrapper to the deep link.
This URL first takes a user to a web page that can redirect the user to the appropriate app store that enables [Deferred Deep Linking](https://en.wikipedia.org/wiki/Mobile_deep_linking#Deferred_deep_linking).
Building a platform to support this level of deep linking isn't something that I will do as a solo dev, so I naturally went looking for a free solution and found two options:
- [Branch.io](https://branch.io/)
- [Firebase Dynamic Links](https://firebase.google.com/docs/dynamic-links)

Both platforms have support for Unity, so I originally decided to go with Branch.io, as the documentation seemed a bit friendlier to me.
I was able to get links working for iOS and Android in a satisfactory way, but I was never able to get macOS and Windows links.
So I gave up on Branch.io and switched to Firebase Dynamic Links.

And I was able to get working deep links for both mobile and desktop!
It just wasn't straight-forward or easy, so I've decided to write this blog post to talk about it.

## The Implementation

I won't go into all the details of setting up Firebase Dynamic Links for your Unity project, as the [Firebase docs](https://firebase.google.com/docs/dynamic-links) do a better job than I could.
Instead, I will just highlight the details of my personal implementation, starting with the Unity code:
```csharp
        private void Start()
        {
#if !UNITY_WEBGL
            Debug.Log("CardGameManager::Start:CheckDeepLinks");
            CheckDeepLinks();
#endif
        }
        
        private void CheckDeepLinks()
        {
            Debug.Log("Checking Deep Links...");
#if UNITY_IOS
            Debug.Log("Should use Firebase Dynamic Links for iOS...");
            FirebaseApp.CheckAndFixDependenciesAsync().ContinueWithOnMainThread(task =>
            {
                var dependencyStatus = task.Result;
                if (dependencyStatus != DependencyStatus.Available)
                {
                    Debug.LogError("Error with Links! Could not resolve all Firebase dependencies: " + dependencyStatus);
                    Messenger.Show("Error with Links! Could not resolve all Firebase dependencies: " + dependencyStatus);
                    return;
                }

                DynamicLinks.DynamicLinkReceived += OnDynamicLinkReceived;
                Debug.Log("Using Firebase Dynamic Links for iOS!");
            });
            return;
#elif UNITY_ANDROID
            if (string.IsNullOrEmpty(Application.absoluteURL))
            {
                DynamicLinks.DynamicLinkReceived += OnDynamicLinkReceived;
                Debug.Log("Using Firebase Dynamic Links for Android!");
            }
#else
            Application.deepLinkActivated += OnDeepLinkActivated;
            Debug.Log("Using Native Deep Links!");
#endif

            if (string.IsNullOrEmpty(Application.absoluteURL))
            {
                Debug.Log("No Start Deep Link");
                return;
            }

            if (!Uri.IsWellFormedUriString(Application.absoluteURL, UriKind.RelativeOrAbsolute))
            {
                Debug.LogWarning("Start Deep Link malformed: " + Application.absoluteURL);
                return;
            }

            Debug.Log("Start Deep Link: " + Application.absoluteURL);
            OnDeepLinkActivated(Application.absoluteURL);
        }

#if UNITY_ANDROID || UNITY_IOS
        private void OnDynamicLinkReceived(object sender, EventArgs args)
        {
            Debug.Log("OnDynamicLinkReceived!");
            var dynamicLinkEventArgs = args as ReceivedDynamicLinkEventArgs;
            var deepLink = dynamicLinkEventArgs?.ReceivedDynamicLink.Url.OriginalString;
            if (string.IsNullOrEmpty(deepLink))
            {
                Debug.LogError("OnDynamicLinkReceived::deepLinkEmpty");
                Messenger.Show("OnDynamicLinkReceived::deepLinkEmpty");
            }
            else
                OnDeepLinkActivated(deepLink);
        }
#endif

        private void OnDeepLinkActivated(string deepLink)
        {
            Debug.Log("OnDeepLinkActivated!");
            var autoUpdateUrl = GetAutoUpdateUrl(deepLink);
            if (string.IsNullOrEmpty(autoUpdateUrl) ||
                !Uri.IsWellFormedUriString(autoUpdateUrl, UriKind.RelativeOrAbsolute))
            {
                Debug.LogError("OnDeepLinkActivated::autoUpdateUrlMalformed: " + deepLink);
                Messenger.Show("OnDeepLinkActivated::autoUpdateUrlMalformed: " + deepLink);
            }
            else
                StartCoroutine(GetCardGame(autoUpdateUrl));
        }

        private static string GetAutoUpdateUrl(string deepLink)
        {
            Debug.Log("GetAutoUpdateUrl::deepLink: " + deepLink);
            if (string.IsNullOrEmpty(deepLink) || !Uri.IsWellFormedUriString(deepLink, UriKind.RelativeOrAbsolute))
            {
                Debug.LogWarning("GetAutoUpdateUrl::deepLinkMalformed: " + deepLink);
                return null;
            }

            if (deepLink.StartsWith(Tags.DynamicLinkUriDomain))
            {
                var dynamicLinkUri = new Uri(deepLink);
                deepLink = HttpUtility.UrlDecode(HttpUtility.ParseQueryString(dynamicLinkUri.Query).Get("link"));
                Debug.Log("GetAutoUpdateUrl::dynamicLink: " + deepLink);
                if (string.IsNullOrEmpty(deepLink) || !Uri.IsWellFormedUriString(deepLink, UriKind.RelativeOrAbsolute))
                {
                    Debug.LogWarning("GetAutoUpdateUrl::dynamicLinkMalformed: " + deepLink);
                    return null;
                }
            }

            var deepLinkDecoded = HttpUtility.UrlDecode(deepLink);
            Debug.Log("GetAutoUpdateUrl::deepLinkDecoded: " + deepLinkDecoded);
            var deepLinkUriQuery = new Uri(deepLinkDecoded).Query;
            Debug.Log("GetAutoUpdateUrl::deepLinkUriQuery: " + deepLinkUriQuery);
            var autoUpdateUrl = HttpUtility.ParseQueryString(deepLinkUriQuery).Get("url");
            Debug.Log("GetAutoUpdateUrl::autoUpdateUrl: " + autoUpdateUrl);

            return autoUpdateUrl;
        }
```

This code is part of a `CardGameManager.cs` MonoBehaviour, which is guaranteed to only run once when the app first starts.
You'll see that there's a lot of platform-specific code, as there are implementation differences between iOS vs Android vs desktop.
I'm also working on a web version of my app, but I don't yet have that working, hence the `#if !UNITY_WEBGL` around the `CheckDeepLinks()`.
The details of the Android and iOS implementation can be derived from the Firebase Dynamic Links docs, so I'd like to focus on the desktop implementation.

The first thing to note is that the code path for desktop is actually using the pure Unity implementation with `Application.deepLinkActivated += OnDeepLinkActivated;`.
I already mentioned that the native deep link implementation has a problem where it relies on custom URI schemas that don't work if the user doesn't already have the app installed.
So how does my solution work on desktop if the user doesn't already have my app installed?

The answer lies in the "Dynamic Links" part of Firebase Dynamic Links.
Note that it isn't called "Firebase Deep Links", as Firebase actually makes a distinction between it's dynamic links and traditional deep links.
Specifically, it treats each "dynamic link" as a web page that then redirects the user to an appropriate "deep link", depending on the platform of the user.
If Firebase detects the user is on iOS or Android, that deep link is an appropriate implementation for that platform.
However, if the user is on a non-mobile platform, the deep link is actually a link to another web page.

In my case, I make use the fact that the user is redirected to another web page.
I can redirect the user to my website, which would then provide further options for redirection.
Effectively, I've built multiple layers of links/redirects, and I think the best way to understand it is with an example that describes all those layers.

## An Example

Let's take a look at the standard CGS Deep Link: [https://cgs.link/standard](https://cgs.link/standard)

1) Layer 1 of the CGS Deep Link is the Firebase Dynamic Link itself: [https://cgs.link/standard](https://cgs.link/standard)

This is where Firebase Dynamic Links will redirect as appropriate for the platform. On iOS or Android, this would mean skipping to layer 3. But on desktop, it will be a redirect to layer 2.

2) Layer 2 is a redirect to my website: [https://www.cardgamesimulator.com/link%3Furl%3Dhttps://www.cardgamesimulator.com/games/Standard/Standard.json](https://www.cardgamesimulator.com/link%3Furl%3Dhttps://www.cardgamesimulator.com/games/Standard/Standard.json)

Here's where the secret of my implementation for desktop lies.

I originally wanted to set up a redirection page at `https://www.cardgamesimulator.com/link?url=<CGS AutoUdate Url>`, but due to issues with url-encoding, I wasn't quite able to get it to get that form.
Instead, the page is more like `https://www.cardgamesimulator.com/<gibberish>`, which will cause my web server to redirect to the 404 page.

So I hacked the 404 page.

I added this code to the body of the 404 page:
```html
  <body onload="updateLinkAndUrl()">
    <script>
    updateLinkAndUrl = () => {
        const path = location.pathname;
        console.log(path);
        let address = decodeURIComponent(path);
        if (path == "/link") {
            address = decodeURIComponent(location.search);
        } else {
            address = address.substring(address.indexOf('?'));
        }
        console.log(address);
        const parameterList = new URLSearchParams(address);
        const map = new Map();
        parameterList.forEach((value, key) => {
            map.set(key, value);
        });
        const autoUpdateUrl = map.get("url");
        const deepLink = "cardgamesim://link?url=" + encodeURIComponent(encodeURIComponent(autoUpdateUrl));
        document.querySelectorAll("pre").forEach((block) => {
          if (navigator.clipboard) {
            let button = document.createElement("button");
            button.innerText = "Copy";
            button.addEventListener("click", async event => {
              const innerButton = event.srcElement;
              await navigator.clipboard.writeText(autoUpdateUrl);
              innerButton.innerText = "Copied!";
              setTimeout(()=> {
                innerButton.innerText = "Copy";
              },1000);
            });
            block.appendChild(button);
          }
        });
        document.getElementById("autoupdateurl").innerHTML = autoUpdateUrl;
        document.getElementById("deeplink").href = deepLink;
    }
    </script>
```

Basically, this edits the page so that it provides a click-able link for layer 3, and a copy button for layer 4.
I encourage users to download my app from the Microsoft Store or the Mac App Store, so that they can then click the layer 3 link.
However, there will be users who prefer Steam, and they would need to launch my app through Steam and then manually enter the layer 4 url.
For those users, I provide a copy button as a slight quality of life improvement.

3) Layer 3 is the native deep link: [cardgamesim://link?url=https%253A%252F%252Fwww.cardgamesimulator.com%252Fgames%252FStandard%252FStandard.json](cardgamesim://link?url=https%253A%252F%252Fwww.cardgamesimulator.com%252Fgames%252FStandard%252FStandard.json)

This native deep link will cause my app to open and automatically enter the information from layer 4 to take the user to the appropriate content within the app.

4) Layer 4 is the CGS AutoUpdate Url: [https://www.cardgamesimulator.com/games/Standard/Standard.json](https://www.cardgamesimulator.com/games/Standard/Standard.json)

This AutoUpdate Url indicates where my app can download all the card game information to display within the app.
If a user chose to download my app from Steam (which doesn't support deep linking), they would have to take some manual steps to enter this url.
My app then takes the json from this url and uses it to display the appropriate content, but the details of that are beyond the scope of this blog post.

## Conclusion

I don't know if anyone else would want to set up a complicated system of deep links like I have done here, but I'm hoping this all this information is at least interesting to someone out there!
