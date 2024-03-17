**Category:** Reverse Engineering
**Flags:** 7
**Difficulty:** 7/10

This challenge was an interesting look into PyInstaller internals and forced me to utilise IDA in ways I hadn't thought of before. I had a great time with this one.

On first execution, the .exe presented some ASCII-art and a standard password validation prompt.
![[qut_ctf_password_validation.png]]

and when I try to enter a non-valid password I get:
![[qut_ctf_prompt_failed.png]]
So let's try and find the correct password (or at least break things enough to not need one).

(I missed this at the time, but was later surprised to find that after attempting to SIGINT the program with ctrl-c, the name of the executing file was revealed as the first flag! "flag{WAAAAHELLOHOWAREYOU}")
![[qut_ctf_first_flag.png]]

To start, my first step is always to try extracting strings to look for hints or anything useful.
Running `strings | grep -i password` came back empty, but when I modified the pattern to 'flag' I was given the second: "flag{youtriedtoohard!!}"
![[qut_ctf_second_flag.png]]
After a few more searches I decided my next step would be disassembling the binary to try to locate the password validation method and attempt to exploit or circumvent it.

After opening the file in IDA64 and examining a few of the decompiled functions, however, I realised it wouldn't be that simple. 

The methods I was returned seemed to be performing a lot of complex memory shifts and executing bytes directly from the stack through function pointers in a very unusual way. Some of the logging strings  and 'flag' keywords returned in the previous step hinted at the reason:![[qut_ctf_die_summary.png]]
The PE was packed with PyInstaller.

At the time, I was largely unfamiliar with PyInstaller stub configuration, and the methods to reverse them. I knew the unpacked binary must be hiding somewhere within the .exe, however, and decided to try and find where this pivotal execution occurred.

Going back to main(), I went through each of invoked methods looking for anything interesting. While the first three seemed to be boilerplate setups, the fourth was the largest by far and had some interesting strings and API calls:

Referencing PyInstaller archives (.pyc files containing decompilable bytecode):
![[qut_ctf_cannot_open_archive.png]]
Searching for libraries and configuring the include path:
![[qut_ctf_setdlldir.png]]

Until finally I found the meat of the application:
![[qut_ctf_create_proc.png]]
A nested function containing a call to the CreateProcess() WinAPI. The function was spawning a child process and halting operations within the main thread. This child was most likely where the actual program was executing from.

Using `Get-ProcessList` I determined the PID of the child process and attached IDA to it. After opening the debugger session and rendering a list of strings from memory, things were noticeably different. 

Filtering for "password" returned the prompts from the input prompt!
![[qut_ctf_password_strings.png]]
I was even given another flag! "flag{NANDOS}" (shoutout Nando's)

But my thirst for flags had only just begun. I changed my filter to "flag{" and that's when I hit the motherload:
![[qut_ctf_motherload.png]]
Three new flags:
- "flag{finalflagLOLILOVENANDOS!}"
- "flag{didyoufindthefile}"
- "flag{IMANOOBLOL}"
Bringing my total to 6 out of 7.

From the surrounding text I gathered that, outside of being hardcoded into the binary, these flags were dispersed as a .log file, a .py file, and as the result of correctly validating with the password prompt.

This just left one flag, and it was the most roundabout by far. 
I searched the rest of the strings, but found nothing. I even tried encoding "flag" as a byte sequence and searching for occurrences separated by null-bytes: 
![[qut_ctf_flag_bytes.png]]
but got nothing.

I decided it was finally time to unpack the executable and retrieve the source code.

Through Google I found the [pyinstxtractor](github.com/extremecoders-re/pyinstxtractor) utility (though I later decided on [pyinstxtractor-ng](https://github.com/pyinstxtractor/pyinstxtractor-ng) because older versions wouldn't run on Python 3.10). The tool wraps Decompyle++ to decompress and extract .pyc archives from PyInstaller .exe stubs.

Of course things couldn't be that simple, however, and I found that the tool encountered an exception when attempting to decompile the binary:
![[qut_ctf_pyinstxtractor_exception.png]]
The ValueError complained that the filename it was attempting to write .pyc data to contained a nullbyte (or 0x00). Nullbytes are considered illegal in filenames so Python will cause an Exception instead of writing the .pyc file.

The fix I found for this was to locate the functions responsible for this exception and manually sanitise the filenames before they could be written to or read from.

The Exception traceback warned that the ValueError was originating from a function called \_writePyc which was responsible for taking decompressed .pyc archive data and writing it to externally .pyc files. 

I added a quick string.replace to exclude to remove any nullbytes contained in a prospective filename:
BEFORE:
![[qut_ctf_pyinstxtractor_fix_before.png]]

AFTER:
![[qut_ctf_pyinstxtractor_fix_after.png]]
(I also added the same fix to the \_fixBarePycs function, the method responsible for prefixing the newly created .pyc files with magic number bytes that identified them as Python bytecode archives)

And after running it again...
![[qut_ctf_pinstxtractor_FIXED.png]]
It worked!
I could now view a directory of the decompiled contents, and the main .pyc archive that had been giving me such a headache.

Extracted as `''$'\001''flag{youtriedtoohard!!}.pyc'`, I decided to rename it to `flag.pyc` to make things simpler. With the archive extracted, I could simply run it through [pycdc](https://github.com/zrax/pycdc) and pipe that to a new .py file to retrieve the source code: `pycdc flag.pyc | tee main.py`

![[qut_ctf_source_code.png]]
And we had the source!

Now we could finally check out the password validation function:
![[qut_ctf_pwd_func.png]]
Oh... so anything works as long as it's 10 characters long and has a 'w' in the second char. I'd call it insecure, but I couldn't figure it out on my own so what do I know? XD

That aside, I was here for the last flag. And after looking around, I figured out why I couldn't see it before.
![[qut_ctf_connection_func.png]]
The program was performing a GET request to an external DNS endpoint: 
`nor.betterwayelectronics.com.au/dfir.php?1337`
and returning the encoded response. 

After some more reading I found that the response data was being passed to another function that checked the content of the `flag{didyoufindthefile}.log` file (from before) to determine if the binary had been run 10 or more times before returning another flag and the data.

To simplify things for myself, however, and avoid having to re-run the file over and over, I just used `curl` to retrieve what I estimated was the flag encoded in some manner: ![[qut_ctf_final_flag_encoded.png]]

I felt I recognised the encoding style, so with a little base64 magic, I decoded the final flag:
![[qut_ctf_funny_pipes.png]]
And that's that!

Overall, this was an extremely interesting challenge that covered a lot of interesting areas in beginner reverse engineering. I definitely chose the difficult method more than once, but I had a great time either way and would recommend anyone looking to mess with their brain a bit to check this challenge out. 

And if you actually made it through this whole writeup, thanks! I definitely spent way too long writing it. XD