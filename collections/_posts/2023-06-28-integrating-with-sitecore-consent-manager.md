---
layout: post
title: Integrating with Sitecore Consent Manager using Next.js
date: 2023-06-29
authors: ["Damien Morrison"]
categories: ["Sitecore", "Sitecore XP", ]
tags: ["Blog"]
description: How to inform Sitecore of a user's tracking consent using Next.js
thumbnail: "assets/images/unsplash-BfrQnKBulYQ-640x427.jpg"
image: "/assets/images/unsplash-BfrQnKBulYQ-2400x1600.jpg"
---

Sitecore has introduced an API for managing the tracking consent with version 10.0, see [https://doc.sitecore.com/xp/en/developers/102/sitecore-experience-platform/manage-a-contact-s-tracking-consent-choices.html](https://doc.sitecore.com/xp/en/developers/102/sitecore-experience-platform/manage-a-contact-s-tracking-consent-choices.html) . You no longer have to fiddle around with the code yourself to somehow remove the SC_ANALYTICS_GLOBAL_COOKIE or update your Cookie Consent Management Tool (Cookiebot, Usercentrics, OneTrust, etc) with the users' selection. Now you simply call the Consent Manager API and Sitecore takes care of the rest. One small disadvantage is that Sitecore then stores the consent in a new cookie SC_TRACKING_CONSENT. An eye for an eye trade, but this cookie 

Sounds easy enough, but how do the integrations to the API look in different setups? 

> This blog post focuses on a headless Sitecore XP setup so it won't be relevant for XM Cloud. It's definitely food for thought how this would be done in XM Cloud. Sitecore must be exposing their own API for these scenarios?? A blog post for another day. 

### Sitecore Controller

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
- Determine the click events or callbacks that you need to hook into in the Cookie Management tool and how to determine if a user has consenting to tracking
- Register a callback or click event on the **client**
- Call the appropriate give or revoke endpoint according to the users' selection
- Add a rewrite entry in the next.config.js to proxy the request from the node server to Sitecore. You could also write a custom route handler if you need to any other processing, i.e. logging.

### Step 1: Handle user's tracking choice

Cookiebot provides decent enough [developer documentation](https://www.cookiebot.com/en/developer) where we can find everything we need. 
-  To react to the user's selection we'll use the *CookiebotOnAccept* and *CookiebotOnDecline* event listeners. 
- To determine if they have consented to tracking, we need to check their selection for Statistics, since this is most appropriate for the type of tracking that Sitecore does. To do this, we'll check the *consent.statistics* property which is added to the global namespace on the client. 

### Step 2: 

### Step 3: 

### Step 4: 


### Headless with Next.js


If you see this error from the Next.js Server, it's most likely a trusted connection issue, since your local server runs unsecured on http and Sitecore on SSL with https. Since your website will (hopefully)  be running on SSL this should only be a development problem. I've read that setting NODE_TLS_REJECT_UNAUTHORIZED to 0 in Node should get around this but it wasn't enough for me. The easiest thing to do was to set up a http binding for my local Sitecore instance. 

```bash
- error Error: unable to verify the first certificate
    at TLSSocket.onConnectSecure (node:_tls_wrap:1540:34)
    at TLSSocket.emit (node:events:513:28)
    at TLSSocket._finishInit (node:_tls_wrap:959:8)
    at ssl.onhandshakedone (node:_tls_wrap:743:12) {
  code: 'UNABLE_TO_VERIFY_LEAF_SIGNATURE'
```




