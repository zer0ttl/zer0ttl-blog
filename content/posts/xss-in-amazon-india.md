---
title: "Reflected XSS in Amazon India"
date: 2019-09-10T16:18:18+05:30
draft: false
description: "A report for a reflected cross site scripting vulnerability(XSS) discovered on [datavault.amazon.in](https://datavault.amazon.in). A URL parameter was reflected back on the page without any sanitization which resulted in reflected XSS on the page. This was reported to Amazon over a year ago and this vulnerability has been fixed since."
author: "zer0ttl"
tags: ["xss", "bug bounty", "web hacking"]
---

# tldr;

A report for a reflected cross site scripting vulnerability(XSS) I discovered on [datavault.amazon.in](https://datavault.amazon.in). A URL parameter was reflected back on the page without any sanitization which resulted in reflected XSS on the page. This was reported to Amazon more than a year ago and this vulnerability has been fixed since.

# Recon :

A list of subdomains for `amazon.in` was gathered using `amass`. `Aquatone` was used to screenshot every domain from the sub-domain list. After combing through the screenshots, I stumbled upon `datavault.amazon.in`.

As I was already authenticated to my `amazon.in` account, this was what I was greeted with when I visited [datavault.amazon.in](https://datavault.amazon.in)

![Unauthenticated screenshot](/img/amazon-india/authenticated.png)

Other than the email address that I used to register with amazon.in, there was nothing interesting on the page. Upon inspecting the source of the page, some interesting javascript files were discovered.

![Unauthenticated screenshot](/img/amazon-india/view_source_javascript_files.png)

Out of the javascript files, the file **installationCommons.js** contained some interesting functions, namely `downloadAffordabilityDocument()` and `downloadDocument()`.

* Function : `downloadAffordabilityDocument()`

     ![Function affordabilityDownloadReport](/img/amazon-india/installationCommons.js_function_downloadAffordabilityDocument.png)

* Function : `downloadDocument()`

     ![Function downloadDocument](/img/amazon-india/installationCommons.js_function_downloadDocument.png)

I was able to extract two endpoints from these functions.

`https://datavault.amazon.in/affordabilityDownloadReport?DocumentName=<DOCUMENT_NAME>`

* Screenshot for the first URL:

     ![affordabilityDownloadReport](/img/amazon-india/affordabilityDownloadReport.png)

`https://datavault.amazon.in/data/DownloadReport?DocumentName=<DOCUMENT_NAME>&RequestType=<REQUEST_TYPE>`

* Screenshot for the second URL:

     ![DownloadReport](/img/amazon-india/downloadReport.png)

Here you can see the the URL parameter **RequestType** is reflected back on the page. This presented me with the opportunity to test HTML injection.

# Testing for HTML injection

Payload :

`<b>BOLD</b>cd`

URL :

`https://datavault.amazon.in/data/DownloadReport?DocumentName=123&RequestType=<b>BOLD</b>cd`

Output :

![DownloadReport](/img/amazon-india/datavault.amazon.in_html_injection.png)

As you can see the user input is not sanitized and is reflected back on the page as HTML code. The `<b>BOLD</b>` was injected to the page. I could inject arbitrary HTML code onto the page. Time to use the `<script>` tags.

# Testing for XSS

Payload :

`<script>alert('XSS');</script>`

URL :

`https://datavault.amazon.in/data/DownloadReport?DocumentName=123&RequestType=<script>alert('XSS');</script>`

Output :

![DownloadReport](/img/amazon-india/datavault.amazon.in_XSS.png)

# Exploitation and Impact

As this is a case of reflected XSS, a specific URL with payload can be crafted and sent to an unsuspecting user. Upon clicking the link, user can either be directed to a malicious website or might be asked to input their password using HTML injection.

Cookies that define authenticated state on amazon.in had the **HTTPOnly** and **Secure** flag set. So cookie stealing was not possible.

# Reporting

* Reported to Amazon Security on 1st March 2018.
* Response from Amazon Security within 24 hrs.
* Mail for 'fix deployed' received on 12th March 2018.

Unfortunately Amazon.in did not have a bug bounty program or a hall of fame at the time, so this report is all that is I have!

If you find a bug with any of the Amazon Inc. services, you can report it to [security@amazon.com](mail:security@amazon.com). AWS has a bug bounty program of its own. You can read about it [here.](https://aws.amazon.com/security/vulnerability-reporting/)
