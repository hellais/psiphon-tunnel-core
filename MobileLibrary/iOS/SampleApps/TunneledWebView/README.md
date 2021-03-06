# iOS Library Sample App: TunneledWebView

## Tunneling UIWebView

*Note: this approach does not work with WKWebView (see [http://www.openradar.me/17190141](http://www.openradar.me/17190141)).*

This app tunnels UIWebView traffic by proxying requests through the local Psiphon proxies created by [PsiphonTunnel](https://github.com/Psiphon-Labs/psiphon-tunnel-core/tree/master/MobileLibrary/iOS/PsiphonTunnel).
The listening Psiphon proxy ports can be obtained via TunneledAppDelegate delegate callbacks (see `onListeningSocksProxyPort` and `onListeningHttpProxyPort` in `AppDelegate.swift`).

This is accomplished by registering `NSURLProtocol` subclass `JAHPAuthenticatingHTTPProtocol` with `NSURLProtocol`.
`JAHPAuthenticatingHTTPProtocol` is then configured to use the local Psiphon proxies.
This is done by setting the [connectionProxyDictionary](https://developer.apple.com/documentation/foundation/nsurlsessionconfiguration/1411499-connectionproxydictionary?language=objc) of [NSURLSessionConfiguration](https://developer.apple.com/documentation/foundation/nsurlsessionconfiguration).
See [`+ (JAHPQNSURLSessionDemux *)sharedDemux`](https://github.com/Psiphon-Labs/psiphon-tunnel-core/blob/c9c4834fba5e7a8b675c3ae493ac17b5975ab0fb/MobileLibrary/iOS/SampleApps/TunneledWebView/External/JiveAuthenticatingHTTPProtocol/JAHPAuthenticatingHTTPProtocol.m#L157) in `JAHPAuthenticatingHTTPProtocol.m`.

We use a slightly modified version of JiveAuthenticatingProtocol (https://github.com/jivesoftware/JiveAuthenticatingHTTPProtocol), which in turn is largely based on [Apple's CustomHTTPProtocol example](https://developer.apple.com/library/content/samplecode/CustomHTTPProtocol/Introduction/Intro.html). 

## *\*\* Caveats \*\*\*

### Challenges

***NSURLProtocol is only partially supported by UIWebView (https://bugs.webkit.org/show_bug.cgi?id=138169) and in 
some versions of iOS audio and video are fetched out of process in mediaserverd and therefore are
not intercepted by NSURLProtocol.***

*In our limited testing iOS 9/10 leak and iOS 11 does not leak.*

### Workarounds

***It is worth noting that this fix is inexact and may not always work. If one has control over the HTML being rendered and resources being fetched with XHR it is preferable to alter 
the media source URLs directly beforehand instead of relying on the javascript injection trick.***

***This is a description of a workaround used in the [Psiphon Browser iOS app](https://github.com/Psiphon-Inc/endless) and not of what is implemented in TunneledWebView.
TunneledWebView *does NOT* attempt to tunnel all audio/video content in UIWebView. This is only a hack which allows tunneling
audio and video in UIWebView on versions of iOS which fetch audio/video out of process.***

#### Background
In [PsiphonBrowser](https://github.com/Psiphon-Inc/endless) we have implemented a workaround for audio and video being 
fetched out of process.

[PsiphonTunnel's](https://github.com/Psiphon-Labs/psiphon-tunnel-core/tree/master/MobileLibrary/iOS/PsiphonTunnel/PsiphonTunnel)
HTTP Proxy also offers a ["URL proxy (reverse proxy)"](https://github.com/Psiphon-Labs/psiphon-tunnel-core/blob/631099d086c7c554a590b0cb76766be6dce94ef9/psiphon/httpProxy.go#L45-L70) 
mode that relays requests for HTTP or HTTPS or URLs specified in the proxy request path. 
 
This reverse proxy can be used by constructing a URL such as `http://127.0.0.1:<proxy-port>/tunneled-rewrite/<origin media URL>?m3u8=true`.

When the retrieved resource is detected to be a [M3U8](https://en.wikipedia.org/wiki/M3U#M3U8) playlist a rewriting rule is applied to ensure all the URL entries
are rewritten to use the same reverse proxy. Otherwise it will be returned unmodified.

#### Fix

* Media element URLs are rewritten to use the URL proxy (reverse proxy).
* This is done by [injecting javascript](https://github.com/Psiphon-Inc/endless/blob/b0c33b4bbd917467a849ad8c51a225c2d4dab260/Endless/Resources/injected.js#L379-L408) 
into the HTML [as it is being loaded](https://github.com/Psiphon-Inc/endless/blob/b0c33b4bbd917467a849ad8c51a225c2d4dab260/External/JiveAuthenticatingHTTPProtocol/JAHPAuthenticatingHTTPProtocol.m#L1274-L1280) 
which [rewrites media URLs to use the URL proxy (reverse proxy)](https://github.com/Psiphon-Inc/endless/blob/b0c33b4bbd917467a849ad8c51a225c2d4dab260/Endless/Resources/injected.js#L319-L377).
* If a [CSP](https://en.wikipedia.org/wiki/Content_Security_Policy) 
is found in the header of the response, we need to modify it to allow our injected javascript to run.
  * This is done by [modifying the
CSP](https://github.com/Psiphon-Inc/endless/blob/b0c33b4bbd917467a849ad8c51a225c2d4dab260/External/JiveAuthenticatingHTTPProtocol/JAHPAuthenticatingHTTPProtocol.m#L1184-L1228) 
to include a nonce generated for our injected javascript, which is [included in the script tag](https://github.com/Psiphon-Inc/endless/blob/b0c33b4bbd917467a849ad8c51a225c2d4dab260/External/JiveAuthenticatingHTTPProtocol/JAHPAuthenticatingHTTPProtocol.m#L1276).

*Requests to localhost (`127.0.0.1`) should be [excluded from being proxied](https://github.com/Psiphon-Labs/psiphon-tunnel-core/blob/master/MobileLibrary/iOS/SampleApps/TunneledWebView/External/JiveAuthenticatingHTTPProtocol/JAHPAuthenticatingHTTPProtocol.m#L283-L287) so the system does not attempt to proxy loading the rewritten URLs. They will be correctly proxied through PsiphonTunnel's reverse proxy.*

## Configuring, Building, Running

The sample app requires some extra files and configuration before building.

### Get the framework.

1. Get the latest iOS release from the project's [Releases](https://github.com/Psiphon-Labs/psiphon-tunnel-core/releases) page.
2. Extract the archive. 
2. Copy `PsiphonTunnel.framework` into the `TunneledWebView` directory.

### Get the configuration.

1. Contact Psiphon Inc. to obtain configuration values to use in your app. 
   (This is requried to use the Psiphon network.)
2. Make a copy of `TunneledWebView/psiphon-config.json.stub`, 
   removing the `.stub` extension.
3. Edit `psiphon-config.json`. Remove the comments and fill in the values with 
   those received from Psiphon Inc. The `"ClientVersion"` value is up to you.

### Ready!

TunneledWebView should now compile and run.

### Loading different URLs

Just update `urlString = "https://freegeoip.net"` in `onConnected` to load a different URL into `UIWebView` with TunneledWebView.

## License

See the [LICENSE](../LICENSE) file.
