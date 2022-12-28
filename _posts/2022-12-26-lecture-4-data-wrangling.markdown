---
layout: post
title: "Lecture 4: Data Wrangling"
date: 2022-12-26 08:35:23 -0500
published: true
---

### Missing Semester IAP 2020

## Data Wrangling
A broad term for changing data format and content with various tools, the process of going from unstructured data like a log file to something like a list of usernames or IP addresses, something useful.

## SSH tricks
Given a SSH server profile `tsp` , let's grab some raw data:  
`ssh tsp journalctl | grep ssh | grep "Disconnected from"`   

From a network traffic perspective, we don't  need anything we're discarding with `grep`.  

In the above command, we're sending the entire `journalctl` entry from our server. To run the `grep` pipes remotely, single-quote the whole ordeal and pipe it into `less`.  

`ssh tsp 'journalctl | grep ssh | grep "Disconnected from"' | less`  

## `sed` stuff
Assuming we have now taken the SSH data and wrote it to a local file `ssh.log` , let's remove everything before "Disconnected from" with `sed`:

  
`cat ssh.log | sed 's/.*Disconnected from//' | less`  
- `s` is a substitute expression which takes two arguments, search, and replace. Here we're saying "Find the following pattern (\*Disconnected from) and replace it with blank".  
- While `sed` is a stream editor, `awk` is a column-based stream processor  

## RegEx
`\.\*`  in the above `sed` command is a regular expression  
- dot means any single character
- star means zero or more of *that* character
- i.e. this boils down to: Select zero or more of any character preceeding the literal string "Disconnected from" , and replace it with "blank"

### Examples & Breakdowns
#### `echo 'aba' | sed 's/[ab]//'`  
- output: `ba`  
- Replace (the first occurance of) any character that is either "a" or "b" with blank
- Default `sed` regex applies the matching and replacement once per line
	- `echo 'aba' | sed 's/[ab]//g'`
	- you can provide the `g` modifier to repeat this as many times as it matches  

#### `echo 'abcaba' | sed -E 's/(ab)*//g'`  
output: `czc`  
- Select zero or more of the string "ab" and replace them with "blank"
- The `-E` flag makes `sed` interpret regular expressions as modern rather than basic
- If you wanted to do this with Basic Regular Expressions (BRE) you'd escape the parentheses with backslashes.
	- `echo 'abcaba' | sed 's/\(ab\)*//g'`  

#### `echo 'abcababc' | sed -E 's/(ab|bc)*//g'`  
output: `cc`  
- Anything that matches "ab" or "bc", replace with blank

#### `echo 'abcabbc' | sed -E 's/(ab|bc)*//g'`  
output: `c`  
- Removing an "a" in the input string  

#### `cat ssh.log | sed -E 's/^.*Disconnected from (invalid )?user .* [0-9.]+ port [0-9]+$//'`  
- output: 
- `^` matches the beginning of a line
- `$` matches the end of a line
	- These two are known as "anchoring" the regular expression (i.e. this regular expression has to match the complete line)
- `(string )?` match zero or one  

#### `ssh myserver journalctl | grep sshd | grep "Disconnected from" | sed -E 's/.*Disconnected from (invalid |authenticating )?user (.*) [^ ]+ port [0-9]+( \[preauth\])?$/\2/' | sort | uniq -c`  
- `sort` will, well, sort its input
- `uniq -c` will collapse consecutive lines that are the same into a single line, prefixed with a count of the number of occurrences

#### `ssh myserver journalctl  | grep sshd  | grep "Disconnected from"  | sed -E 's/.*Disconnected from (invalid |authenticating )?user (.*) [^ ]+ port [0-9]+( \[preauth\])?$/\2/'  | sort | uniq -c  | sort -nk1,1 | tail -n10`  
- `sort -n` will sort in numeric (instead of lexicographic) order  
	- **lexicographic**: (also known as **lexical order**, or **dictionary order**) is a generalization of dictionary alphabetic order to sequences of ordered symbols
- `-k1,1` means “sort by only the first whitespace-separated column”  
	- `1,1` here are the arguments saying "I want to start at the 1st column, and end at the 1st column"
- The `,n` part says “sort until the `n`th field, where the default is the end of the line (sorting by the whole line doesn't matter in this case)
- Printing the last (`-n`) 10 lines with `tail` is optimal here since sort by default outputs in ascending order, meaning the largest values are going to be at the bottom

#### `ssh myserver journalctl  | grep sshd  | grep "Disconnected from"  | sed -E 's/.*Disconnected from (invalid |authenticating )?user (.*) [^ ]+ port [0-9]+( \[preauth\])?$/\2/'  | sort | uniq -c  | sort -nk1,1 | tail -n10  | awk '{print $2}' | paste -sd,`  
- Creating a .csv of the most common usernames used to login
- `awk` by default will take in whitespace separated columns as arguments and allow you to perform operations on that data  
- `paste` takes a bunch of lines and pastes them into a single line based on a given delimiter (comma in this case)  
- `awk` , also being an entire programming language in its own, can process regex statements (see next)  

#### `| awk '$1 == 1 && $2 ~ /^c[^ ]*e$/ { print $2 }' | wc -l`  
- Print the number of single-use usernames that start with "c" and end with "e"

#### `awk` is a programming language
- You can do the same thing as above in `awk` syntax:
```
BEGIN { rows = 0 }
$1 == 1 && $2 ~ /^c[^ ]*e$/ { rows += $1 } 
END { print rows }
```
- `BEGIN` is a pattern that matches the start of the input (and `END` matches the end)
- Now, the per-line block just adds the count from the first field (although it’ll always be 1 in this case), and then we print it out at the end  
- ref: [https://backreference.org/2010/02/10/idiomatic-awk/](https://backreference.org/2010/02/10/idiomatic-awk/)  

### Greedy vs Non-greedy Expressions
- RegEx expressions like dot star (.\*) are greedy, meaning they will match as much as *they can*  
- Example: `echo 'Disconnected from invalid user Disconnected from 84.211' | sed 's/.*Disconnected from //'`   
	- output: `84.211`  
	- In this scenario, since the username we're trying to extract ("Disconnected from") matches until the second occurance, it is replaced with blank as dot star is a greedy expression  

### RegEx Symbol Cheatsheet-ish
-   `.` means “any single character” except newline
-   `*` zero or more of the preceding match
-   `+` one or more of the preceding match
-   `[abc]` any one character of `a`, `b`, and `c`
-   `(RX1|RX2)` either something that matches `RX1` or `RX2`
-   `^` the start of the line
-   `$` the end of the line
-   adding a suffix of `?` changes the greed of a given expression

### RegEx Debugger
- RegEx can get really complicated and layered in a single command, debugger can be a great way to break down more complicated expressions
- Example: [https://regex101.com/r/qqbZqh/2](https://regex101.com/r/qqbZqh/2)  

### Capture Groups
- For if you want to capture a value and store it for later use, in the case of our log file, we want to capture the username
- Any parentheses-enclosed value is technically a capture group i.e. `(invalid |authenticating )`
- But, they won't necessarily do anything until you refer back to them in the replacement
- Example: `sed -E 's/.*Disconnected from (invalid |authenticating )?user (.*) [^ ]+ port [0-9]+( \[preauth\])?$/\2/'`  
	- output: `root\n ftp_test\n ...`  
	- Above, the second capture group is being referred to in the replacement ("\2") and thus the value will be output  

### RegEx Black Hole
- This stuff gets out of hand, especially for matching email:
	- [https://www.regular-expressions.info/email.html](https://www.regular-expressions.info/email.html)  
	- [https://emailregex.com](https://emailregex.com) 
	- [https://stackoverflow.com/questions/201323/how-can-i-validate-an-email-address-using-a-regular-expression/1917982#1917982](https://stackoverflow.com/questions/201323/how-can-i-validate-an-email-address-using-a-regular-expression/1917982#1917982) 
	- [https://fightingforalostcause.net/content/misc/2006/compare-email-regex.php](https://fightingforalostcause.net/content/misc/2006/compare-email-regex.php)  
	- [https://mathiasbynens.be/demo/url-regex](https://mathiasbynens.be/demo/url-regex)  

### Analyzing the Data
`| paste -sd+ | bc -l`  
- Concatenate all the values with a plus sign and pipe them into `bc` for more calculation capabilities
- `bc` - (basic calcaulator) an arbitrary-precision calculator language that can read from STDIN

#### `R`  
`ssh myserver journalctl  | grep sshd  | grep "Disconnected from"  | sed -E 's/.*Disconnected from (invalid |authenticating )?user (.*) [^ ]+ port [0-9]+( \[preauth\])?$/\2/'  | sort | uniq -c  | awk '{print $1}' | R --no-echo -e 'x <- scan(file="stdin", quiet=TRUE); summary(x)'`  
- R is wierd, but good at analysis and [plotting](https://ggplot2.tidyverse.org)   
- Here we created a vector comprised of the input stream of numbers, and ran the `summary()` function on it

#### `gnuplot`  
`ssh myserver journalctl  | grep sshd  | grep "Disconnected from"  | sed -E 's/.*Disconnected from (invalid |authenticating )?user (.*) [^ ]+ port [0-9]+( \[preauth\])?$/\2/'  | sort | uniq -c  | sort -nk1,1 | tail -n10  | gnuplot -p -e 'set boxwidth 0.5; plot "-" using 1:xtic(2) with boxes'`  
- If you just want simple plotting capabilities, gnuplot is probably easier

#### `xargs`  
`rustup toolchain list | grep nightly | grep -vE "nightly-x86" | sed 's/-x86.*//' | xargs rustup toolchain uninstall`  
- If you're looking to install or remove things on your system  from a longer list, `xargs` plus the data wrangling we've seen already can be a powerful tool
- Above we see a command to uninstall old nightly builds of Rust by extracting the old build names and then passing them via `xargs` to the uninstaller

### Wrangling Binary Data
`ffmpeg -loglevel panic -i /dev/video0 -frames 1 -f image2 -  | convert - -colorspace gray -  | gzip  | ssh mymachine 'gzip -d | tee copy.jpg | env DISPLAY=:0 feh -'`  
- Command to use ffmpeg to capture an image from our camera, convert it to grayscale, compress it, send it to a remote machine over SSH, decompress it there, make a copy, and then display it

### Resources
- [https://regexone.com](https://regexone.com) 
- [https://regex101.com](https://regex101.com) 
- Especially useful for Mac users looking for data sources for the exercises: [https://eclecticlight.co/2018/03/21/macos-unified-log-3-finding-your-way/](https://eclecticlight.co/2018/03/21/macos-unified-log-3-finding-your-way/)  

### Public Data Sets
- [https://stats.wikimedia.org/EN/TablesWikipediaZZ.htm](https://stats.wikimedia.org/EN/TablesWikipediaZZ.htm)
- [https://ucr.fbi.gov/crime-in-the-u.s/2016/crime-in-the-u.s.-2016/topic-pages/tables/table-1](https://ucr.fbi.gov/crime-in-the-u.s/2016/crime-in-the-u.s.-2016/topic-pages/tables/table-1)
- [https://www.springboard.com/blog/data-science/free-public-data-sets-data-science-project/](https://www.springboard.com/blog/data-science/free-public-data-sets-data-science-project/)