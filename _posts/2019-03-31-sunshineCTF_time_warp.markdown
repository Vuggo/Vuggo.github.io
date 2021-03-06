---
layout: blogpost
title:  "sunshineCTF - Time Warp"
date:   2019-03-31 21:00:00 +2100
categories: ctf
---

## Challenge: 
TimeWarp
## Description: 
Oh no! A t3mp0ral anoma1y has di5rup7ed the timeline! Y0u'll have to 4nswer the qu3stion5 before we ask them!<br>
nc tw.sunshinectf.org 4101
<br>
## Solution:
When you connect to the server it gives you a prompt saying<br>
"I'm going to give you some numbers between 0 and 999. Repeat them back to me in 30 seconds or less!"<br>
so clearly this is a scripting challenge.

I manually input a few numbers to see how it responded and the server sent an error along with the correct number upon failure so a script that keeps trying to connect and updating the input file was necessary. Initially I was manually making a small input file for testing and if there were only correct numbers but not every correct number then the connection would halt and eventually close meaning we needed a placevalue number in our file (which i chose as 1) although I should have chose -1 because the number range was [0,999].<br>
Here is my solution 
<br>

```bash
#!/bin/bash
while :
do

    nc tw.sunshinectf.org 4101 < in.txt > out.txt
   
    #sometimes the first line returned was the next needed number 
    #and sometimes it was the error message so valA and valB handle both of those cases
   
    valA=$(tail -2 out.txt | head -1 | wc -c) 
    valB=$(tail -1 out.txt | wc -c)
    
    #gets only the line count value of the wc function and not also the trailing filename string
    inLen=$(wc -l in.txt | awk '{print $1}') 
    
    #the size of both values must be fewer than 5 bytes so we check to make sure one of the 
    #two is the actual next number we need. 
    
    #If it is then we take our original input file (which should have a trailing placeholder 1 at the moment)
    #and write to a temporary file with everything except for that 1. Next it will append the value we need
    #which is stored in the out.txt file (which is the server response file) onto the temporary file.
    
    #Finally it will append 1 to the temporary file again and then change the file name to in.txt.
   
    if [ $valA -lt 5 ]; then
        head -n -1 in.txt > temp.txt
        tail -2 out.txt | head -1 >> temp.txt
        echo 1 >> temp.txt
        mv temp.txt in.txt
        cat in.txt
    elif [ $valB -lt 5 ]; then
        head -n -1 in.txt > temp.txt
        tail -1 out.txt >> temp.txt
        echo 1 >> temp.txt
        mv temp.txt in.txt
        cat in.txt
    fi

done
```
After running the script there is a point where the 1 no longer gets replaced and the connection halts. Soon it closes and the bottom of our out.txt file contains the lines<br> 
"Wow! You did it!<br>
As reward for fixing the timestream, here's the flag:<br>
sun{derotser_enilemit_1001130519}"<br>