# SMS Receiver API

## Introduction

Developers use phone numbers for many aspects of building an application:

* account identifier (especially in emerging markets where email usage is low)
* social graph (based on phone number contact list for example)
* communication channel (i.e. call the user, send text message, etc.)
* anti-abuse signal (phone numbers are limited, may require physical identity verification in some places)
* multi-factor auth (e.g. 2-step verification with SMS OTP)
* account recovery (e.g. look up a forgotten account, as a password reset option)

The challenge is that usage of phone numbers for these purposes typically requires proof that a user currently controls the phone number (phone number verification), and existing verification mechanisms on the web are cumbersome, requiring users to manually input one-time verification codes. Easing this has been a long standing feature request for the web from many of the largest global developers.

There are a variety of ways to verify control over phone numbers, but a randomly generated one-time passcode (OTP) sent by SMS to the number in question is the most common. Presenting this code back to developer’s server proves control of the phone number. In this proposal, we focus on the ability to programmatically obtain one-time codes from SMS as solution to ease the friction and failure points of manual user input of SMS codes, which is prone to error and phishing.

  * goal: make the most [common existing](#scenarios) phone number verification flow (SMS OTP) match ease of use in native apps.
  * non-goal: this proposal does not attempt to move developers off of existing phone number use cases and verification mechanisms, nor does it cover obtaining the phone number itself. Specifically, the developer must have previously obtained the phone number via existing mechanisms (e.g. user form input, autofill, etc.), which aren't addressed in this proposal.

## Prior Art

There are two comparable APIs that we should use as a reference.

First, the native [Android API](https://developers.google.com/identity/sms-retriever/overview) is an **imperative** API that gives access to the **full** contents of the SMS message. Here is what it looks like:

```java
// Starts SmsRetriever, which waits for ONE matching SMS message until timeout
// (5 minutes). The matching SMS message will be sent via a Broadcast Intent with
// action SmsRetriever#SMS_RETRIEVED_ACTION.
SmsRetrieverClient client = SmsRetriever.getClient(this /* context */);
Task<Void> task = client.startSmsRetriever();
```

In order to use the native SMS retrieval mechanism, the SMS message content must be formatted appropriately, with a hashcode derived from the native app package and cert fingerprint. For example:

```
<#> Your ExampleApp verification code is: 123ABC78
FA+9qCX9VSu
```

Secondly, Safari on iOS has a declarative [autocomplete](https://developer.apple.com/documentation/security/password_autofill/enabling_password_autofill_on_an_html_input_element) API that provides an integration with the native keyboard. iOS applies heuristics to extract OTPs from SMSes to pass it back to the `<input>` element. Here is what it looks like:

```html
<input autocomplete="one-time-code"/>
```

## Proposal

The following is an early exploration / baseline of what this API could look like. We expect them to change drastically as we learn more about the space.

From a UX perspective, we want to get out of the way as much as possible from the web author, while still keeping users aware and in control of what's going on.

<img src="mock3.png" width="250px">

To support this user flow, we propose two complementary API components:

* an imperative client-side [javascript API](#imperative-api)
* a [formating convention](#formatting) for SMS messages

The former gives web pages a mechanism to receive SMSes and the latter is used as a mechanism to make sure that the origin boundaries are kept without additional mediation / gesture from the user. 

You can find here other [alternatives](#alternatives-considered) under consideration.

### Imperative API

In this formulation, browsers provide an imperative API to request the contents of an incoming SMS. Here is one possible formulation / shape, based on Android’s [SMS Retriever API](https://developers.google.com/identity/sms-retriever/overview): 

```javascript
if (navigator.sms) {
  alert("feature not available :(");
  return;
}
try {
  let {content} = await navigator.sms.receive();
  alert("sms received! " + content);
} catch (e) {
  alert("time out!");
}
```

Some corner cases are covered [here](#Spec).

There are a couple of nice side effects of the imperative API:

* first, it can derive the [declarative API](#Declarative-API) (but not otherwise)
* second, it can offer a spectrum of mediation / consent / permissions / interventions without any re-activation of the ecosystem (e.g. a [permission-less UX](#automatic-ux), a [non-blocking UX](#unblocking-ux) and an [autofill UX](#autofill-ux))

### Declarative API

An interesting implication of uncovering the lower level imperative API is that it can derive the high level declarative API without any loss of (a) browser mediation and (b) graceful degradation.

Here is an example of a custom element that can be embedded in pages to polyfill existing deployments of the declarative autofill API:

```javascript
/**
 *  <sms-receiver> is a custom element that extends <input> elements
 *  with an autocomplete="one-time-code" with the imperative
 *  navigator.sms.receive() API. Submits the form when it receives
 *  the SMS.
 * 
 *  Example:
 *
 *  <form>
 *    <input is="sms-receiver" 
 *           name="otp" 
 *           regex="\s([A-Za-z0-9]{6})\." 
 *           autocomplete="one-time-code" 
 *           placeholder="Code (6 digits)" 
 *           required />
 *    <input type="submit" />
 *  </form>
 *
 *  Parameters:
 *
 *    - regex: a regular expression used to parse the contents of the
 *             sms message and get the OTP code.
 *
 *
 *  Degrades gracefully to the autofill UI or manual input when the
 *  API is not available.
 *
 */
customElements.define("sms-receiver",
  class extends HTMLInputElement {
    connectedCallback() {
      this.receive();
    }
    async receive() {
      try {
        let {content} = await navigator.sms.receive();
        let regex = this.getAttribute("regex");
        let code = new RegExp(regex).exec(content);
        if (!code) {
          return;
        }
        this.value = code[1];
        this.form.submit();
      } catch (e) {
        console.log(e);
      }
    }
  }, {
    extends: "input"
});
```

### Formatting

In this proposal, to support the isolation between different origins (without extra user mediation), we define a formatting convention in the SMS message that enables them to be addressed to a specific origin and routed by the browser securely:

```
Your OTP is: 123ABC78.
For: https://example.com
```

Long term, we expect the formatting to be browser agnostic, but while GMS core releases are still rolling out, Android still needs an [app hash](https://developers.google.com/identity/sms-retriever/verify#computing_your_apps_hash_string) to know which APK it should redirect the SMS to. There is an interesting trick we could do to combine URLs with App Hashes, embedding them as URL parameters (making them valid android SMSes as well as valid web urls, which we can use to derive origins):

```
Your OTP is: 123ABC78.
For: https://code.sgo.to?hash=s3LhKBB0M33
```

In this formulation, the last few characters (e.g. `s3LhKBB0M33`) are used to route the SMS from Android to the Browser APK and the origin is used to route from the Browser process to the right requesting tab. 

Another nice side effect of this formulation is that the URL could be used as a fallback mechanism in case anything fails (e.g. poor mobile network reception leads to an SMS being delivered many hours/days later).

```
Your OTP is: 123ABC78.
For: https://code.sgo.to/verify.php?otp=123ABC78&hash=s3LhKBB0M33
```

The `For` footer should point at a `HTTPS` or `localhost` URL.

## Security

From a security perspective, the biggest consideration with this API is crossing an origin boundary, which we believe is mitigated by the [formatting](#formatting) addressing scheme.

This API is also **only** available via `https` or `localhost` (for development purposes). We don't entirely adopt the concept of [trustworthy urls](https://www.w3.org/TR/powerful-features/#is-url-trustworthy) because it covers more schemes (e.g. `data://123`) than we would like to (our initial intuition is that (a) `https` and `localhost` covers most cases and (b) it needs to be clear to the user what public facing entity its sending the SMS). 

This API is also **only** available on main frames, to avoid abuse by third party iframes / libraries.

## Privacy

From a privacy perspective, there are a few considerations to be taken:

- inner frames and ad networks
- fingerprinting
- awareness and control

The first concern is somewhat easy to address: we propose the API should be **unavailable outside of top level frames**.

The second and the third concerns are hard to be talked about abstractly, outside of a specific [UX formulation](#UX). We believe, however, that under the proposed UX formulation, the following attack vectors are addressed.

### User Tracking

Phone numbers are an effective stable identifier for a user that enable cross-site and online/offline tracking. Obtaining the phone number is the point at which user is typically (or should be) asked for consent and best educated about the implications of sharing this information, so is not addressed in this proposal.

However, the ability to verify the user’s phone number automatically in the background via an SMS retrieval API is a mechanism by which ongoing presence of the user could be determined, at least on a particular device where the developer already knows the phone number. Existing Android APIs mitigate this by allowing existing SMS notifications and SMS history to continue to be visible to the user, giving them insight that this may be happening (and providing a disincentive for a service to “guess” the user’s phone number, spam the number, and try to detect if this user is present by seeing if retrieval works). In practice, in the Android native app ecosystem, this hasn’t been found this to be a vector for abuse, especially given the cost of sending SMS and the visibility of the attack to users.

### Phishing

SMS OTP are readily phishable and an existing widespread concern. While not making this worse, this proposal attempts to mitigate by avoiding and lessening the occurrence of users to manually enter OTP (so as to be less conditioned to phishing, and/or more conscious of where they enter OTP), and by making the OTP only available via programmatic mechanism to the intended recipient (i.e. by specifying the target origin in the message contents).

## Annex

### Related APIs

#### Credential Management and WebAuthn APIs 

CM API and WebAuthn facilitate alternative forms of authentication. CM API facilitates interaction with password credentials and WebAuthn allows developers to interact with authentication hardware that provides much stronger, phishing resistant, and more usable multi-factor authentication on the web. While better alternatives for authentication, these APIs do not provide any communication mechanism or reputation signals that developers also use phone numbers for, so are not a comprehensive alternative to phone number or SMS OTP. 

#### Notifications

Browser notification APIs provide a communication channel to developers, but developers often still prefer or also request a verified phone numbers since it checks that the user is reachable at this number and facilitates voice communication and reasonably real-time two-way communication on practically any time of mobile phone (no dependence on OS or version, pretty much all phones can handle SMS). 

#### OAuth etc.

OAuth and similar protocols allow developers to obtain information such as verified phone number from an identity provider (IDP). However, this relies on user having provided, verified, and maintain their phone number with the IDP, and be comfortable using this model to share data (e.g. knowing their usage of 3p services). The IDPs themselves (unless an authoritative provider for a phone number, such as a carrier), need to verify phone number ownership, and typically still use SMS OTP for that purpose. 

#### reCAPTCHA

Captcha APIs provide an alternative anti-abuse signals (i.e. that user is not a bot) that developer sometime rely on phone numbers for (in that phone numbers are often limited and require human involvement to procure, and as such as hard to produce at scale).

#### Payment APIs

Phone numbers are sometimes used for carrier billing schemes. Payment APIs offer an alternative as well as signal of user quality (having a payment instrument often involves identity verification and ability / history of being able to pay).

### Alternatives Considered

#### UX

There are many different UX formulations that are under consideration, with different trade-offs between user awareness, control and friction. Here are the ones we are exploring:

##### Automatic UX

<img src="mock2.gif" width="250px">

##### Unblocking UX

<img src="mock3.png" width="250px">

##### Autofill UX

<img src="mock4.png" width="250px">

##### Opt-In UX

<img src="mock6.gif" width="250px">

##### Opt-Out UX

<img src="mock.gif" width="250px">


#### Heuristic Autofill

In addition to autofill annotations, the browser could also heuristically extract and autofill OTP, with user confirmation and without explicit developer support. 

However, getting access to SMS without coordination from the developer (i.e. without explicit formatting of the SMS or indication that developer is expecting an SMS) will be a challenge, as browser would require ongoing access to SMS. 

Note that iOS provides heuristic-based OTP autofill, but iOS provides browser / keyboard access to SMS; but on Android, browsers may not have or even be able to request indefinite SMS access.

#### Phone Number Assertion API

If phone number has already been verified for a given device or user account, browser could return a verifiable assertion of the phone number. 

```
// This is just a draft/example of what a API could look like.
let phone = await navigator.credentials.get({phone: true});
verify(phone);
```

This could be implemented by having developer interact with identity providers (IDPs), which have already verified and are aware of the user’s phone number, and could vouch for this information, in the same way Google Sign-In and similar federated identity flows currently work for email addresses. 

Several identity providers such as Facebook (Account Kit), Truecaller, and others already provide APIs like this verified phone numbers.

#### OTP Retrieval API

Provide a higher level API for obtaining OTP, which could be provided by a variety of transport mechanisms (email, time-based authenticator apps running on the device, not just SMS)

```
// This is just a draft/example of what a API could look like.
let otp = await navigator.credentials.get({otp: true});
verify(otp);
```

### Spec

There are a couple of alternatives that we considered from an API shape perspective. First, subclassing `EventTarget`:

```javascript
let receiver = new SMSReceiver();
receiver.addEventListener("receive", ({content}) => {
  console.log(content);
});
receiver.receive();
```

But it seemed awkward since this is a one-shot event.

The other formulation that we think is worth noting is to use a static method outside of `navigator`. Example:

```javascript
let {content} = SMSReceiver.receive();
```

This could work equally well compared to `navigator.sms.receive()`, but would pollute the global namespace rather than the `navigator` namespace.

The other consideration here is whether to enable aborting the request or maybe pass a custom timeout:

```javascript
let abort = new AbortController();
setTimeout(() => {
  // abort after two minutes
  abort.abort();
}, 2 * 60 * 1000);
  
try {
  let {content} = await navigator.sms.receive(abort);
} catch (e) {
  // deal with errors
}
```
