---
layout: post
title: Integrating with Sitecore Consent Manager using Next.js
date: 2023-06-29
authors: ["Damien Morrison"]
categories: ["Sitecore", "Sitecore XP", "Next.js"]
tags: ["Blog"]
description: How to inform Sitecore of a user's tracking consent using Sitecore XP and Next.js
thumbnail: "assets/images/unsplash-BfrQnKBulYQ-640x427.jpg"
---

Sitecore introduced the [Tracking Consent API](https://doc.sitecore.com/xp/en/developers/102/sitecore-experience-platform/manage-a-contact-s-tracking-consent-choices.html) for managing the tracking consent since version 10.0. You no longer have to fiddle around with the code yourself to  remove the SC_ANALYTICS_GLOBAL_COOKIE. Now you simply call the Consent Manager API and Sitecore takes care of the rest. One small disadvantage is that Sitecore then stores the consent in a new cookie SC_TRACKING_CONSENT. Not quite an eye for an eye as this cookie merely stores the user's consent choice and it isn't used to identify the user in any way. 

Sounds easy enough, but how do the integrations to the API look in different setups? 

>This blog post focuses on a headless Sitecore XP setup so it won't be relevant for XM Cloud. 

It's definitely food for thought how this would be done in XM Cloud. Sitecore must be exposing their own API for these scenarios? A blog post for another day.

## Sitecore Integration

On the side of Sitecore, it's a very painless integration with Sitecore's Consent Manager. See [Sitecore's documentation](https://doc.sitecore.com/xp/en/developers/102/sitecore-experience-platform/manage-a-contact-s-tracking-consent-choices.html) for a full run down, but essentially you just need to call the *GiveConsent* and *RevokeConsent* methods from the Consent Manager Service respectively. 

Sending through null to these calls will instruct Sitecore to use the *IContentStorage* service, which, by default, is the new cookie mentioned above, SC_TRACKING_CONSENT. 

```c#
[RoutePrefix("api/trackingconsent")]
public class TrackingConsentController : Controller
{
    private readonly IConsentManager _consentManager;

    public TrackingConsentController(IConsentManager consentManager)
    {
        this._consentManager = consentManager;
    }

    [HttpPatch]
    [Route("give")]
    public ActionResult GiveConsent()
    {
	    this._consentManager.GiveConsent(null);

        return new JsonResult()
        {
            Data = new { Message = "Consent has been granted" }
        };
    }

    [HttpPatch]
    [Route("revoke")]
    public ActionResult RevokeConsent()
    {
        this._consentManager.RevokeConsent(null);

        return new JsonResult
        {
	        Data = new { Message = "Consent has been revoked" }
        };
    }
}
```

That's pretty much all we need to do. Sitecore handles the rest internally. 

In this case of GiveConsent, Sitecore will start (or continue) tracking and will update the ContentStorage (the SC_TRACKING_CONSENT cookie by default). 

You can check that it's working by inspecting the response headers on the request. You should see two cookies being set. 

| Key        | Value                                                                                                                                                           |
|------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Set-Cookie | SC_TRACKING_CONSENT=W3siU2l0ZU5hbWUiOiJjb3Jwb3JhdGVfZGUiLCJJc0NvbnNlbnRHaXZlbiI6dHJ1ZX1d0; expires=Mon, 27-Jun-2033 14:09:43 GMT; path=/; secure; SameSite=None |
| Set-Cookie | SC_ANALYTICS_GLOBAL_COOKIE=c7253135ff3e4535a941684b684531ef\|False; expires=Mon, 27-Jun-2033 14:09:43 GMT; path=/; secure; HttpOnly; SameSite=None              |

In the case of Revoke, Sitecore needs to carry out its due diligence to ensure any tracking is stopped and any trace of that contact is removed. Equally important, is that the SC_ANALYTICS_GLOBAL_COOKIE is removed. 

This again can be checked by inspecting the headers. This time you will also see two headers, but the SC_ANALYTICS_GLOBAL_COOKIE should be expired (to a time in the past). The browser will then pick up the expiry date and remove it. 

| Key        | Value                                                                                                                                                           |
|------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Set-Cookie | SC_TRACKING_CONSENT=W3siU2l0ZU5hbWUiOiJjb3Jwb3JhdGVfZGUiLCJJc0NvbnNlbnRHaXZlbiI6dHJ1ZX1d0; expires=Mon, 27-Jun-2033 14:09:43 GMT; path=/; secure; SameSite=None |
| Set-Cookie | SC_ANALYTICS_GLOBAL_COOKIE=03af44f45566487c94f27b1de72618a9\|False; expires=Thu, 29-Jun-2023 15:26:18 GMT; path=/; secure; HttpOnly; SameSite=None               |

At the time of writing it is the 30th of June 2023 and we can can see the expiry date of the Global Analytics cookie has been set to a day earlier on the 29th, hence it is removed.

## Cookiebot Integration in Next JS

Once the Give/Revoke endpoints are in place we will need to integrate with the Cookie Management Tool so that the appropriate endpoint is called when the user makes their selection. 

I will demonstrate the integration with Cookiebot, however the approach will be the same for all tools. 
- Determine the click events or callbacks that you need to hook into in the Cookie Management tool and how to determine if a user has consented to tracking
- Register a callback or click event on the **client**
- Call the appropriate give or revoke endpoint according to the user's selection
- Add a rewrite entry in the next.config.js to proxy the request from the node server to Sitecore. You could also write a custom route handler if you need to any other processing, i.e. logging.

### Handle user's tracking choice

Cookiebot provides decent enough [developer documentation](https://www.cookiebot.com/en/developer) where we can find everything we need. 
-  To react to the user's selection we'll use the *CookiebotOnAccept* and *CookiebotOnDecline* event listeners. 
- To determine if they have consented to tracking, we need to check their selection for Statistics, as this category describes the type of tracking that Sitecore does. To do this, we'll check the *consent.statistics* property which is added to the global namespace on the client. 

### Register the click event in Next.js

To register the client event in Next.js, we can use the useEffect React hook. Be sure to pass an empty array as a dependency to useEffect. We don't need anything other than a one-time registration here.  We can also hook into cleanup with the return function to remove the event listeners.

```js
useEffect(() => {
	const cookieBotAcceptEvent = async () => {
		await handleCookieBotAccept(currentSite);
	};
	  
	const cookieBotDeclineEvent = async () => {
		await handleCookieBotDecline(currentSite);
	};
	
	if (typeof window !== 'undefined') {
		window.addEventListener('CookiebotOnAccept', cookieBotAcceptEvent);
		window.addEventListener('CookiebotOnDecline', cookieBotDeclineEvent);
	}
	
	return () => {
		window.removeEventListener('CookiebotOnAccept', cookieBotAcceptEvent);
		window.removeEventListener('CookiebotOnDecline', cookieBotDeclineEvent);
	};
}, []);
```

### Handle the Click Events and call the endpoints

Once an event has been triggered, we still need to determine if a consent choice has actually been made for statistics.  A little bit of logic is required to handle the different choices. Ensure to read the documentation on which selection fires which event.  

Once we're finally nutted down what the choice was, we can call the give/revoke endpoints.

```js
const giveConsent = async (site: string): Promise<void> => {
	await fetch(`/api/trackingConsent/give?sc_site=${site}`, {
		method: 'PATCH',
	}).catch((error) => {
		console.error('Error giving consent:', error);
	});
};

const revokeConsent = async (site: string): Promise<void> => {
	await fetch(`/api/trackingConsent/revoke?sc_site=${site}`, {
		method: 'PATCH',
	}).catch((error) => {
		console.error('Error revoking consent:', error);
	});
};
  
export const handleCookieBotAccept = async (site: string): Promise<void> => {
	//is fired on buttons "Allow all cookies" & "Allow individual selection"
	if (window.Cookiebot?.changed && window.Cookiebot.consent.statistics) {
		giveConsent(site);
	} else if 
		(window.Cookiebot?.changed && !window.Cookiebot.consent.statistics) {
			revokeConsent(site);
	}
};
  
export const handleCookieBotDecline = async (site: string): Promise<void> => {
	//is fired on buttons "Withdraw you consent" & "Use necessary cookies only"
	if (window.Cookiebot?.changed && !window.Cookiebot.consent.statistics) {
		revokeConsent(site);
	}
};
```


### Proxy request to Sitecore

The final part of the flow is to ensure that Next.js proxy ferries the request to Sitecore and back. This is a good time to add the SC API Key since the proxy runs server-side.  Here we can add the following entry to the Next.js rewrite module in next.config.js.

```js
{
	source: '/api/trackingConsent/:path*',
	destination: `${jssConfig.sitecoreApiHost}/macaw/api/trackingconsent/:path*?sc_apikey=${jssConfig.sitecoreApiKey}`,
},
```

#### Trusted Connection Issue

If you see this error from the Next.js Server, it's most likely a trusted connection issue, since your local server runs unsecured on http and Sitecore on SSL with https. Since your website will (hopefully)  be running on SSL this should only be a development problem. I've read that setting NODE_TLS_REJECT_UNAUTHORIZED to 0 in Node should get around this but it wasn't enough for me. The easiest thing to do was to set up a http binding for my local Sitecore instance. 

```bash
- error Error: unable to verify the first certificate
    at TLSSocket.onConnectSecure (node:_tls_wrap:1540:34)
    at TLSSocket.emit (node:events:513:28)
    at TLSSocket._finishInit (node:_tls_wrap:959:8)
    at ssl.onhandshakedone (node:_tls_wrap:743:12) {
  code: 'UNABLE_TO_VERIFY_LEAF_SIGNATURE'
```

## Conclusion

As we can see, Sitecore certainly does a lot of heavy-lifting for us here, but there is still come integration work to be done in Next.JS to handle the events and notify Sitecore. A word of warning, there have been reports that Sitecore is not always respecting the consent choice being stored. See the blog post by George Change [here](https://georgechang.io/posts/2023/making-sitecore-xp-compliant-with-gdpr-and-other-privacy-laws-because-its-not/). This is obviously an issue that Sitecore will hopefully sort out soon. 
