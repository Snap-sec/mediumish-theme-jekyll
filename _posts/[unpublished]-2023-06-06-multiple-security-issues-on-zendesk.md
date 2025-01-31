---
layout: post
title:  "Hacking Zendesk - Cache Deception, Privilege Escalation and more"
author: snapsec
categories: [ article,broken-access-control ]
image: assets/images/zendesk/zendesk.png
---


Another expedition to choose a new target to hack at Snapsec stopped at Zendesk. Zendek aligned with most of our testing principles, which we consider while choosing a new target to hack. Their available metrics remarked that the Zendesk security team was responsive and acknowledged the work of security researchers and pentesters without any bias. The application size was vast, and there were hardly any restrictions on the scope of testing. The quest began on Zendesk and the rest(a major part of what we did) will be documented here on this blog.


## What is zendesk?

Zendesk is basically a comprehensive management platform for all customers and deals within a business. The various functionalities of zendesk product are summarised as below:

{{zendesk pic}}

-   Handling customer support operations and offers support by email, mobile, social media and voice with complete data reporting.
-   Integrate to your own websites to create support request.
-   A platform that offers live chat solution and support businesses to increase sales conversation through engaging leads on their websites.
- Integrate other platform's data 
- Call center software offers more personal phone support and enhances enterprise productivity
- User roles within the organization.
- Customized role segregation.

Zendesk provides various business-related services, which include **sales, marketplace, and other services**. The size of the application was massive and the *implemented functionalities were complex as well*. There were a lot of things to learn and hack. So, as usual, we started with the *product documentation*, *available tutorials*, *YouTube channels* and other informative guides. You have an unfair advantage over other hackers if you first understand a product and work with it as a normal customer.

## Our methodology  simpleified:

We have a general methodology with which we approach any target. It covers all, from recon to learning intended behaviours and then finally abusing the flow in various forms. The various principles comprising our methodology are as under:

#### 1. Understanding the purpose of the App

At this point, we will try to figure out what the purpose of the app is. For example, Zendesk is used for management of various processes in sales and business. We will learn it and then try to emulate the process in the app.



#### 2. Understand the App logically


At this point, we look into several features separately and try to understand how they have implemented them. In Zendesk deals, for example, we keep track of the *various stages* a deal goes through in the application, such as *lead, meeting, responses, comments, collaborators*, and so on.


#### 3. Understand how the app technically 

In this phase, we figure out how things work technically for the application. For example, what kind of information is being published about deals/contacts in the zendesk? And does the target app have an access control model or any kind of technology like markdown or other stuff?


#### 4. Creating Assumptions 
 
In this phase, we make several assumptions about lots of things in the app. For example, if the deal is private to a user, can we still send a response via HTTP requests that can change it? Can this deal be made public by sending various http requests to it? We keep a note of all of the generated assumptions so that we don’t miss anything.

#### 5. Attacking the App

At this point, we will use all of our assumptions that can create abnormal behaviour in the app. For example, can we post comments on the private deals of a user? Can we get the contact details of other users?




## Bypassing CSRF using Cache Deception Attack

Cache deception attack is often caused by a non-standard server-side setting overriding recommended cache-control directives. Due to the cache misconfiguration, an attacker may send a specially crafted link to users, which will result in the leak of sensitive data. In this case we were able to bypass **CSRF protection by leveraging the cache deception attack**.

The key component of the attack is "path confusion," which involves altering URL routes to trick the cache server into identifying private HTTP answers as publicly cacheable documents.


For example in Zendesk the URL `https://developer.zendesk.com/account` refers to content containing sensitive data that should not be cached. The attacker tricks the target user into making a request to `https://developer.zendesk.com/account/<Anything>.css` causing the server to respond with response containing sensitive information specific to the victim.
 
The proxy, on the other hand, interprets the request as `https://developer.zendesk.com/account/Anything>.css` as being a request for a non-existent-cacheable ‘.css' file, which in turn causes the sensitive content to be stored in the cache and accessible by others.
 
 Which means an attacker can trick the victim into visiting `https://developer.zendesk.com/account/<Anything>.css` and later visiting the same url to get the cached response of the page which contains the name, email, and CSRF token of the victim. Later, the CSRF TOKEN can be used to perform an application-wide CSRF attack.


__Here is a quick Video POC for the vulnerability:__
 
 
<iframe width="560" height="315" src="https://www.youtube.com/embed/NeAbTaalP9s" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
 
 
---

## Adding and removing tags from restricted contacts,deals and leads

Zendesk Sell is a CRM software focused on empowering sales teams to win more deals. So the main functionality is to generate deals. Deal generation goes through various steps : _from gathering contacts, leads and then the cycle finally ends upon deal generation._
So it is, in fact, a process involving huge sets of data. Zendesk uses the concept of tags to gather a particular data set under a particular tag for ease of management.
The intended behaviour of the application is that a user has read/write access to his tags i.e. the tags he owns. So the user can add/remove/modify the tags he has ownership over.
We identified a vulnerable endpoint through which a user was able to get read/write access on the tags created by other users.

 
- Adding tags to contacts, leads, deals 

 ```http
POST /apis/tags/api/v1/taggings/batch_add.json

{"tagid":"3","dealid":"231"}
```
 
- Removing tags from contacts, leads, deals

 
 ```http
POST /apis/tags/api/v1/taggings/batch_untag.json

{"tagid":"2"}
```


But we are able to add/modify/remove the tags of other users that we are not supposed to have authority over. This could lead to quite a chaotic situation as we can add tags to any unauthorised deal or even render a particular data set(contacts/leads/deals) tag less by removing the tags.


---

## Changing Documents on contacts after admin revokes our access to contacts

In Zendesk, an admin can invite a user to a limited access role through which the user can view, update, delete, and convert their own contacts, leads, and deals in their account. The user can also add documents to their contacts etc. The admin has full authority over invited users. The admin can revoke the ownership of the contacts of the invited user. As a result of this, the invited user is rendered unable to edit the documents attached to the contacts, and the ownership of the documents is transferred to the admin. But we identified a vulnerable endpoint through which we were still able to gain edit access on the documents attached with the contacts even after the admin had revoked our access.

- Editing documents on contacts
 
```http
PUT /apis/uploader/api/v2/uploads/<upload_id>.json 
```
 
So we were able to affect the integrity of the documents as we were able to edit the names of documents and even change their extensions.


---

## Privilege escalations leads to add/remove appointments on restricted contacts.

Since zendesk allows users to collaborate on various functionalities in the app. This was accomplished by assigning different roles to different functions in the app. One role in the app was defined as *limitted permission user*, who has access to limited features in the app. The following user was *unable to create the contacts*, nor could he add any appointments to them.

{{1}}

But we were able to find an API path with *broken access controls*  that led the **limited user** to create and remove appointments with the *contacts* even though this functionality was restricted from the *said role*.

Thus, the limited user was successfully able to exploit this *API request* to create appointments with the contacts just by using their **contact id**.

```http
POST /apis/appointments/api/v1/appointments.json HTTP/1.1
Host: app.futuresimple.com
Connection: close
Content-Length: 255
x-interaction-id: 3c4eee5f-bda4-4581-bd8d-1e54aeb2ac01
Origin: https://app.futuresimple.com
x-parent-operation-id: 3c4eee5f-bda4-4581-bd8d-1e54aeb2ac01
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/73.0.3683.103 Safari/537.36
x-csrf-token-generated-at: 1597815645
Content-Type: application/json
Accept: application/json, text/javascript, */*; q=0.01
X-Requested-With: XMLHttpRequest
x-csrf-token-signature: <VALUE>
Referer: https://app.futuresimple.com/crm/contacts/288149956
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Cookie:<VALUE>

{"appointment":{"contact_id":"<contact id>","start_at":"2020-08-17T00:00:00.000Z","end_at":"2020-08-17T23:59:59.999Z","is_public":false,"send_updates":true,"name":"New Appointment on Sunda","location":"New York","description":"","all_day":true}
```

The limited user was able to cancel/delete all of the appointments via the below `http request` by changing the **appointment id**.

```http
DELETE /apis/appointments/api/v1/appointments/<<apppointment id>>.json HTTP/1.1
Host: app.futuresimple.com
Connection: close
Content-Length: 12
x-interaction-id: 91a95d78-f953-4303-f70c-f36f8f3c222c
Origin: https://app.futuresimple.com
x-parent-operation-id: 91a95d78-f953-4303-f70c-f36f8f3c222c
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/73.0.3683.103 Safari/537.36
x-csrf-token-generated-at: 1597816624
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
Accept: application/json, text/javascript, */*; q=0.01
X-Requested-With: XMLHttpRequest
x-csrf-token-signature: <VALUE>
Referer: https://app.futuresimple.com/crm/contacts/288149956
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Cookie: <VALUE>
platform=web
```
This could lead to *uanuthorized creation of appointments* in an organisation and also could have impacted the *integrity and availability  of the contacts and appointments*.


---

## Exposing restricted documents on private contacts.

As mentioned, the permissions restrict the *various roles* from accessing different features on the account. In this case the permission we were testing for was **limited user permissions,** which restricted the assigned user from accessing the *contacts or documents associated with those contacts*. 

This issue led a user to *access the documents associated with the private contacts of admins*. The admins could associate various documents with their contacts which include *sales scripts, pdf files and other neccessary documents*. 

{{png of douments}}

On analyzing the process, we came to conclusion that an http request was responsible for uploading a document to any *contact*, but the *limited user was unable to exploit this feature to upload documents on private contacts*. We then changed the *above request method* and appended *the contact's id* with the API path and it gave us the *information of the contact in json response*. And in response it leaked the **Public S3 url to access the admin private documents in the json response of this contact**.  We were able to get those *documents without having any permissions on them*.


 ## Unauthorised user is able to leak the sales pipeline and move deals into them


A sales pipeline is a visual illustration of how customers move through the [sales process] stages—from prospecting for leads to making the final sale. It provides valuable insights into each deal, allowing your agents to track their progress, identify any gaps in their process, and form strategies to hit their sales targets.

The admin of the account creates a pipeline and keeps it personal to him without giving any access to the invited user to those pipelines. Admin can move his deals to this pipeline as he controls the ownership of this pipeline.
But we analysed  the following vulnerable request through which we were able to accomplish the task of adding our own deals to admin's pipeline .  

- Moving deals into admin's sales pipeline
```http
PUT /apis/sales/api/v1/deals/<deal-id>.json HTTP/1.1

{"pipeline_stage_id":}
```
So, here due to broken access control on this endpoint we were able to gain an unauthorised access on admin's pipeline allowing us to move our deals into admin's pipeline.





## A simple user in the organisations able to create loss reasons with restricted permissions

When a deal is lost, it is moved into the Lost stage of the sales pipeline. To gain additional information about why this happened, admin can prompt the `Sell users` to provide a reason for why the deal was lost. This prompt is enabled by a user with admin rights. Admin users can also edit and add other loss reasons.
But we were able to accomplish the task of creating loss reasons inside the organisation via the following API path with broken access control.

- Creating a loss reason
```http
POST /apis/sales/api/v1/loss_reasons.json HTTP/1.1

{"loss_reason":{"name":"My reason"}}
```

 

 
That's all for now! If you have any questions, suggestions or requests, please contact us on Twitter([@snap_sec](https://twitter.com/snap_sec))
See you next time.







#### About us

Snapsec is a team of security experts specialized in providing pentesting and other security services to secure your online assets. We have a specialized testing methodology that ensures in-depth testing of your business logic and other latest vulnerabilities. 

 If you are looking for a team that values your security and ensures that you are fully secure against online security threats, feel free to get in touch with us #[support@snapsec.co](mailto:support@snapsec.co)
 
 
