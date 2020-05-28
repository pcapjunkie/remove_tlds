# remove_tlds

NOTE: Its best to view this in RAW format or else youll miss some characters and formatting


Remove TLDs (top level domain) or Public Suffix from a list of domains

I needed a way to remove TLD's from a log that included domain names. a list of domains needed to be cleaned up so it can be used with freq.py. That meant leaving just the unique portion of the domain and removing the TLD's. Unfortunatly I did not have the result needed for the class, but this script is more useful. In the class they just wanted you to drop the last column of the full domain name (rev | cut -d "." -f2- | rev). 

Desired outcome: Remove all portions of the domain name that matches the suffix list (whether that is TLD or the complete public suffix list), and not just the portion of the domain after the last period.

Why this and not existing solutions? 
I started on this and spent a long time learning sed/awk/regex to get this done. Defeat was not an option. Also other solutions use python or perl from what i have seen. if i find a more elegant bash script then ill be happy to switch.

This will be more of a walkthrough since its just a bash script. I have yet to figure out how to make this work using a separate file for the tld/suffix list inside the script. so I have resorted to just storing the results inside the script file itself. If you have different tld/suffix filters/lists then you will have to build separate scripts.

First of all, download a tld or suffix list of choice. In this guide i am using the one from "The Public Suffix List". The public suffix list = "A "public suffix" is one under which Internet users can (or historically could) directly register names. Some examples of public suffixes are .com, .co.uk and pvt.k12.ma.us. The Public Suffix List is a list of all known public suffixes." I felt this list is the most complete.

https://www.publicsuffix.org/list/
https://github.com/publicsuffix/list

Step 1: Download the list

wget https://www.publicsuffix.org/list/public_suffix_list.dat --no-check-certificate

Step 2: extract domain names from your log. you can use commandline kungfu or manually extract them to store in a separate file. there are other resources for parsing and extracting domain names from web/firewall/dns logs

Step 3: get a baseline of what suffixes/TLD's your domaian log has it in. This part is a work in progress. this is just a quick way to see .. yup.. i have 990000000 .com domains in there.

grep "\\." justdomains.log | rev | cut -d "." -f1 | rev | sort | uniq -c | sort -nr | head -10

5861 com
467 net
292 jp
271 ru
243 org
96 de
184 cn
136 br
126 fr
106 in
    
Step 4: prep the tld/suffix file for use with the script, and save the results into a new script file
  #remove the comments (lines that begin with \\) 
    sed '/^\/\//d'
  #remove blank lines 
    sed '/^$/d'
  #remove the astrixes and the connecting period 
    (in the future i need to script just adding these as all possible variables to the other suffixes they are typically used with. until this is done, the script may need to be ran multiple times on your target log file) 
  # here it is with all three bundled together 
    sed '/^\/\//d;/^$/d;s/\*\.//g'
    cat public_suffix_list.dat | sed '/^\/\//d;/^$/d;s/\*\.//g'
  #add period to the beginning of each tld since this list is missing them, and i want to get matches on (period)+(suffix)
    sed 's/^/./'
  #sort the list by length 
    (so that longer tld's are processed before shorter ones, eg .com before .co, so your not just left with an extra "m" for all your .com domains) then select only the tld column so we can drop the size column 
    awk '{print length, $0}' | sort -nr | cut -d " " -f2
  #escape out the periods with backslashes so the TLDs will work with sed regular expression (used later on) with literal periods 
    sed 's/\./\\\\./g'
  #output the cleaned up tld list to a new script 
    > remtld.sh

# the full command to "prep" the script. = 

cat public_suffix_list.dat | sed '/^\/\//d;/^$/d;s/\*\.//g' | sed 's/^/./' | awk '{print length, $0}' | sort -nr | cut -d " " -f2 | rev | sed 's/\./\\\\./g' > remltd.sh

Step 5: edit the script file
#now edit the script and add the beginning and end parts update the filename you want to clean up. note that this script is designed to modify the original file, so make sure you keep a backup.

# i prefer nano, you can use vi/gedit/whatever
nano tldrem.sh 

you will see the suffixes/TLDs in the file (eg \\.com \\.co.tw ). 
add the following to the beginning, BEFORE the TLDs

#!/bin/bash -x
tlds=( 

And then add the following AFTER the TLD list

)

for i in "${tlds[@]}"; do

    sed -i -e "s/${i}$//" justdomains.log;
    
done

---
# A few notes on what this is doing. 

The uses the prepped results of your tld/suffix file and adds them to the variable "tlds". 
then for each item in tlds (each suffix), sed is ran on the source log file specified. 
each row in your source log file is checked for matching suffixes/tlds at the end of each row in your log file. 
if it matches, the tld/suffix is removed. This could be tweaked so that instead of just checking the end of the line, to remove all matches that have a trailing space OR linefeed or whatever (eg sed 's/${i}\s//'). that way the domain name doesnt have to be the last column in the log file. i just havent tested it yet. the important thing is that we dont want matches in the middle of a domain due to ligitimate subdomains. like pcapjunkie.netmon.com , if .net matches then wed be left with pcapjunkiemon, when we wanted pcapjunkie.netmon to be left. however if the domain is followed by a / or space, or tab or linefeed or etc etc, then we can make sed match those special characters only.

Step 6: save your changes, make the script executable

chmod +x tldrem.sh

Step 7: Run it, see what happens

./tldrem.sh
+ for i in '"${tlds[@]}"'
+ sed -i -e 's/\.s3\.dualstack\.ap-southeast-2\.amazonaws\.com$//' justdomains.log
+ for i in '"${tlds[@]}"'
+ sed -i -e 's/\.s3\.dualstack\.ap-southeast-1\.amazonaws\.com$//' justdomains.log
+ for i in '"${tlds[@]}"'
+ sed -i -e 's/\.s3\.dualstack\.ap-northeast-2\.amazonaws\.com$//' justdomains.log
+ for i in '"${tlds[@]}"'
+ sed -i -e 's/\.s3\.dualstack\.ap-northeast-1\.amazonaws\.com$//' justdomains.log

Step 7: check your log file. make sure no "legitimate" domain suffixes/tld's are left

grep "\." justdomains.log | rev | cut -d "." -f1 | rev | sort | uniq -c | sort -nr
      5 wordpress
      5 wix
      5 go
      1 tumblr
      1 squarespace
      1 mihanblog
      1 ipage
      1 68
      1 163

Note: in this case I don't believe any of these remaining domains suffixes/tld's are valid. if i did find some, i would compare the results do the original copy of the log, and figure out if I need to add more TLD's to search for. Optionally you can just re-run the script against the same log and it will remove more. but then you risk cutting out part of the legitimate unique domain names. (such as pcapjunkie.blog.blog, what if "blog" were the actual registered domain?, we would want to keep that, and only drop the suffix/tld)
