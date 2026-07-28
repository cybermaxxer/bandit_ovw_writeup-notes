# overthewire: bandit — full walkthrough (level 0 → 33)

bandit is overthewire's entry level wargame. each level is an ssh login, and solving one gets you the password for the next. there's no real exploitation happening, it's linux fundamentals: file permissions, `find`/`grep` chains, encodings, cron, setuid, and a handful of git and shell escape puzzles toward the end.

i'm not just dumping commands here. every level opens with a small table from my own study notes (what the mechanism actually is, why it applies to that level, what the flags mean), then the actual solve. no passwords, keys, or flags included anywhere, every level just ends with "check `/etc/bandit_pass/banditn`" and i've redacted the value itself.

connect with:

```bash
ssh banditN@bandit.labs.overthewire.org -p 2220
```

port 2220, not 22. bandit runs on a non standard port so it doesn't collide with the host's own ssh service or get buried under the constant bot noise hitting port 22.

## table of contents

- [level 0 → 1: logging in](#level-0--1-logging-in)
- [level 1 → 2: a file named -](#level-1--2-a-file-named--)
- [level 2 → 3: filename with spaces](#level-2--3-filename-with-spaces)
- [level 3 → 4: a hidden file](#level-3--4-a-hidden-file)
- [level 4 → 5: find the readable one](#level-4--5-find-the-readable-one)
- [level 5 → 6: find by exact properties](#level-5--6-find-by-exact-properties)
- [level 6 → 7: find by owner across the whole filesystem](#level-6--7-find-by-owner-across-the-whole-filesystem)
- [level 7 → 8: grep for a keyword](#level-7--8-grep-for-a-keyword)
- [level 8 → 9: the one line that isn't duplicated](#level-8--9-the-one-line-that-isnt-duplicated)
- [level 9 → 10: pulling text out of binary noise](#level-9--10-pulling-text-out-of-binary-noise)
- [level 10 → 11: base64](#level-10--11-base64)
- [level 11 → 12: rot13](#level-11--12-rot13)
- [level 12 → 13: peeling a stack of compression layers](#level-12--13-peeling-a-stack-of-compression-layers)
- [level 13 → 14: ssh key auth](#level-13--14-ssh-key-auth)
- [level 14 → 15: talking to a raw tcp port](#level-14--15-talking-to-a-raw-tcp-port)
- [level 15 → 16: same thing, over tls](#level-15--16-same-thing-over-tls)
- [level 16 → 17: scanning a port range for the real service](#level-16--17-scanning-a-port-range-for-the-real-service)
- [level 17 → 18: diffing two files](#level-17--18-diffing-two-files)
- [level 18 → 19: a .bashrc that logs you out](#level-18--19-a-bashrc-that-logs-you-out)
- [level 19 → 20: your first setuid binary](#level-19--20-your-first-setuid-binary)
- [level 20 → 21: setuid binary that connects back to you](#level-20--21-setuid-binary-that-connects-back-to-you)
- [level 21 → 22: reading a cron job](#level-21--22-reading-a-cron-job)
- [level 22 → 23: cron job with a derived filename](#level-22--23-cron-job-with-a-derived-filename)
- [level 23 → 24: planting your own script in a cron job](#level-23--24-planting-your-own-script-in-a-cron-job)
- [level 24 → 25: brute forcing a 4 digit pin](#level-24--25-brute-forcing-a-4-digit-pin)
- [level 25 → 26: escaping a pager based restricted shell](#level-25--26-escaping-a-pager-based-restricted-shell)
- [level 26 → 27: using the setuid binary you land next to](#level-26--27-using-the-setuid-binary-you-land-next-to)
- [level 27 → 28: password sitting in a git repo](#level-27--28-password-sitting-in-a-git-repo)
- [level 28 → 29: password scrubbed from HEAD, alive in history](#level-28--29-password-scrubbed-from-head-alive-in-history)
- [level 29 → 30: password on a different branch](#level-29--30-password-on-a-different-branch)
- [level 30 → 31: password in an annotated tag](#level-30--31-password-in-an-annotated-tag)
- [level 31 → 32: force pushing past .gitignore](#level-31--32-force-pushing-past-gitignore)
- [level 32 → 33: escaping an uppercase only shell](#level-32--33-escaping-an-uppercase-only-shell)

---

## level 0 → 1: logging in

**concepts**

| concept | explanation | why it matters here |
|---|---|---|
| ssh | encrypted remote login protocol | how you get a shell on the bandit box at all |
| `-p` flag | "port", tells ssh which port to connect on instead of the default 22 | bandit listens on 2220, mandatory here or you get connection refused |
| `user@host` syntax | tells ssh who to authenticate as and where | `bandit0` is the starting account. drop the `user@` part and ssh defaults to your local machine's username, which doesn't exist remotely |

**solve**

```bash
ssh bandit0@bandit.labs.overthewire.org -p 2220
```

first connection throws a host key prompt, type `yes`, then enter the password (`bandit0`, given on the site). the terminal shows nothing while you type it, that's intentional, not broken. once you're in:

```bash
cat readme
```

that's the whole level. it exists purely so "can you connect at all" isn't tangled up with an actual puzzle.

---

## level 1 → 2: a file named `-`

**concepts**

| concept | explanation | why it matters here |
|---|---|---|
| leading dash ambiguity | commands parse a leading `-` as option syntax by default | a file literally named `-` gets misread as a flag instead of a filename |
| `./` prefix | forces the shell to resolve what follows as a relative path, not an option | bypasses the flag parsing entirely, since a path can't be mistaken for a bare flag |
| `--` | posix convention meaning "end of options, everything after is a literal argument" | lets you pass `-` as a filename without needing `./` |

is `./` a real mechanism or just convention? real: `.` is always an alias for the current directory, `/` separates it from what follows. it's a literal path, not a trick.

is `--` universal? it's a widely followed posix convention across most cli tools, not a hard guarantee for every program, but it's safe to reach for by default.

**solve**

```bash
ls -halps
cat *
```

`cat *` prints everything in the current directory. if you want to be more precise, `cat -- -` also works (`--` means skip flag parsing, same idea as `-t` in `ss -t`).

---

## level 2 → 3: filename with spaces

**concepts**

| concept | explanation | why it matters here |
|---|---|---|
| word splitting | the shell splits unquoted input on whitespace before handing it to a command | a filename with spaces gets read as multiple separate arguments unless you stop that splitting |
| quoting | wrapping an argument in quotes tells the shell "treat this whole thing as one token" | the direct fix, no need to escape each space individually |
| backslash escaping | a `\` before a character tells the shell "treat the next character literally" | an alternative to quoting, same result, more typing |

**solve**

```bash
ls -halps
cat 'spaces in this filename'
```

quoting the whole string is generally less error prone than escaping every space by hand.

---

## level 3 → 4: a hidden file

**concepts**

| concept | explanation | why it matters here |
|---|---|---|
| dotfile convention | a filename starting with `.` is treated as hidden | the password file is hidden this way inside `inhere` |
| `-a` flag | "all", tells `ls` to include entries starting with `.` | without it the hidden file never shows up in a plain listing |
| `-l` flag | "long format", shows permissions, owner, size, date | useful for spotting the real file among decoys by size alone |

is the dot for hidden a real filesystem attribute? no, it's pure display convention. `ls` and file managers respect it, but the filesystem itself doesn't treat dotfiles any differently at the permission level.

**solve**

```bash
cd inhere
ls -halps
cat '.hidden'
```

---

## level 4 → 5: find the readable one

**concepts**

| concept | explanation | why it matters here |
|---|---|---|
| `file` command | inspects a file's actual content (magic bytes/headers) and reports its real type | lets you find the one actually ascii/text file among ten binary decoys |
| magic bytes | specific byte sequences at the start of a file that identify its format | this is how `file` "knows" the type, not by extension or name |
| `*` glob | matches every filename in the current directory | expands to every decoy so `file` checks them all in one shot |

is `file` guessing from the name? no, it's a real mechanism, it reads actual bytes and matches them against a signature database. filenames are irrelevant to it.

**solve**

```bash
cd inhere
file ./-file*
```

one line will say `ascii text`, everything else will say `data` or similar.

```bash
cat '-file07'
```

(your file number will vary, check the `file` output, don't assume mine matches)

---

## level 5 → 6: find by exact properties

**concepts**

| flag | meaning | why used here |
|---|---|---|
| `find <path>` | starting point for a recursive search | `inhere` is the directory root, the file is buried in nested subdirectories |
| `-type f` | "type: file" | excludes directories from the results |
| `-size 1033c` | size filter, `c` suffix means bytes | matches the exact 1033 byte requirement given by the level |
| `! -executable` | negates the executable test | filters out decoy files marked executable |

is `c` for bytes derivable or just memorized? memorized, it's `find`'s own unit vocabulary (`c` = bytes, `k` = kb, `m` = mb), not derived from anything. skip the suffix and `find` defaults to 512 byte blocks, giving you silently wrong matches.

multiple predicates in `find` get anded together automatically, no explicit `-a` needed.

**solve**

```bash
find inhere -type f -size 1033c ! -executable
cat <path find gave you>
```

---

## level 6 → 7: find by owner across the whole filesystem

**concepts**

| flag | meaning | why used here |
|---|---|---|
| `find /` | search starting at filesystem root | the target isn't under your home directory this time, it's anywhere on the box |
| `-user` / `-group` | filters by file owner / owning group | the level gives you both, an exact identity match |
| `-size 33c` | exact byte count, same `c` suffix as before | narrows a filesystem wide search down to one file |
| `2>/dev/null` | redirects stderr to the null device, discarding it | searching from `/` as a low privilege user throws constant "permission denied" noise, this silences it |

**solve**

```bash
find / -user bandit7 -group bandit6 -size 33c -type f
cat #(the path here, that actually showed up)
```

---

## level 7 → 8: grep for a keyword

**concepts**

| concept | explanation | why it matters here |
|---|---|---|
| `grep <pattern> <file>` | prints every line matching the pattern | `data.txt` is huge, `grep` scans line by line and only prints matches, no manual scrolling |
| literal string matching | grep by default treats your search term as a literal substring (basic regex) | no regex syntax needed, the word itself is the whole query |
| `-w` flag | "whole word", matches only when the pattern stands alone as a full word | avoids accidental partial matches inside longer words |

**solve**

```bash
cat data.txt | grep 'millionth'
```

`-w` isn't strictly required here since "millionth" is unique enough on its own, but it's a good habit against noisy substring hits in general.

---

## level 8 → 9: the one line that isn't duplicated

**concepts**

| flag/command | meaning | why used here |
|---|---|---|
| `sort` | reorders lines alphabetically | `uniq` only compares adjacent lines, so sorting is what groups duplicates together in the first place |
| `uniq` | removes adjacent duplicate lines by default | alone it just dedupes, doesn't isolate the singletons |
| `uniq -u` | `-u` = "unique", prints only lines with zero duplicates | exactly what the level asks for |
| `\|` pipe | sends one command's stdout into the next command's stdin | chains sort straight into uniq without a temp file |

`uniq -u` (only uniques) and `uniq -d` (only duplicates) are opposites, easy to mix up.

**solve**

```bash
sort data.txt | uniq -u
```

---

## level 9 → 10: pulling text out of binary noise

**concepts**

| concept | explanation | why it matters here |
|---|---|---|
| `strings <file>` | scans a file and prints runs of printable characters above a minimum length | pulls the readable password out of surrounding binary garbage |
| default minimum length | `strings` defaults to sequences of 4+ printable characters | short enough to catch real text, long enough to skip most random noise |
| piping to `grep` | narrows the flood of extracted strings down to a specific anchor | the level tells you the password is preceded by several `=` characters |

catting this file raw risks the same terminal garbling issue as level 4, binary bytes can contain escape sequences your terminal tries to interpret as formatting.

**solve**

```bash
strings data.txt | grep '='
```

---

## level 10 → 11: base64

**concepts**

| concept | explanation | why it matters here |
|---|---|---|
| base64 encoding | maps binary data onto a 64 character printable alphabet (a-z, a-z, 0-9, +, /) | why `data.txt` looks like readable garbage instead of raw binary |
| `base64` command | encodes by default, decodes with a flag | you need the decode direction here |
| `-d` flag | "decode", reverses the encoding back to the original bytes | without it the command tries to re-encode already-encoded text, garbage output |
| `=` padding | fills the gap when original data isn't a clean multiple of 3 bytes | expected and normal at the end of the string, not part of the actual data |

is base64 encryption? no, zero secrets involved. anyone with the string can decode it, it's a format conversion, not a security measure.

**solve**

```bash
cat data.txt
base64 -d data.txt
```

---

## level 11 → 12: rot13

**concepts**

| concept | explanation | why it matters here |
|---|---|---|
| rot13 | shifts each letter 13 places through the alphabet, wrapping around | a becomes n, b becomes o, etc, 13 is exactly half of 26 |
| `tr <set1> <set2>` | "translate", maps each character in set1 to the character in the same position in set2 | you build the rotated alphabet mapping yourself as the two sets |
| `tr 'a-za-z' 'n-za-mn-za-m'` | set1 is the normal alphabet, set2 is that alphabet rotated 13 | n is the 14th letter (13 after a), starting set2 there and wrapping to m covers the full rotation, same logic for lowercase |

why does applying rot13 twice return the original text? 13 + 13 = 26, a full loop around the alphabet, landing back exactly where you started. that's why the same command both encodes and decodes.

**solve**

```bash
cat data.txt
tr 'A-Za-z' 'N-ZA-Mn-za-m' < data.txt
```

this one took me a minute to actually get. the way it works: you give `tr` a set1 with x characters, and a set2 that also adds up to x characters total, possibly built from multiple ranges. here set1 is `A-Z`, 26 chars, and set2 is `N-ZA-M`, also 26 chars split across two ranges. once one range in set2 fills up, the next range picks up the mapping. so `A-Z` maps to `N-ZA-M` (a→n, b→o, ...), and once you hit n in the first set, it wraps and maps to a. same logic repeats for lowercase right after.

---

## level 12 → 13: peeling a stack of compression layers

**concepts**

| concept | explanation | why it matters here |
|---|---|---|
| hexdump | text representation showing each byte as two hex digits | the format `data.txt` is stored in, not directly usable by any decompression tool |
| `xxd -r` | `-r` = reverse, converts a hexdump back into raw binary | the actual inverse operation, undoing the hexdump |
| `file <filename>` | reports the actual detected format | run after every step to learn what layer comes next (gzip, bzip2, tar, etc) |
| scratch `/tmp` dir | a throwaway writable directory outside your home | avoids clutter across many decompression steps and avoids collisions with other players on the shared box |

is `xxd -r` self inverse like rot13? no, different pair. `xxd` (no `-r`) goes binary to hex text, `xxd -r` goes hex text back to binary, a true inverse pair, not a self inverse.

**solve**

```bash
mkdir /tmp/$(whoami)-work && cd /tmp/$(whoami)-work
cp ~/data.txt .
xxd -r data.txt layer1
file layer1
```

loop from here: identify the format with `file`, rename if the tool insists on a matching extension, decompress, check again.

```bash
mv layer1 layer1.gz
gzip -d layer1.gz
file layer1
# repeat until file reports plain ascii text
cat #the thing
```

---

## level 13 → 14: ssh key auth

**concepts**

| concept | explanation | why it matters here |
|---|---|---|
| private key file | the secret half of a keypair, must stay protected | handed to you directly instead of a password this time |
| `-i` flag | "identity file", tells ssh which private key to use for authentication | without it ssh falls back to its own default keys, which won't match |
| key permission check | ssh refuses to use a private key that's readable by other users | real security mechanism, a leaked world readable key lets anyone impersonate you |
| `chmod` | changes file permission bits | needed to lock the key down before ssh will accept it |

**solve**

```bash
ls -la
scp -P 2220 bandit13@bandit.labs.overthewire.org:sshkey.private .
chmod 600 sshkey.private
ssh -i sshkey.private bandit14@bandit.labs.overthewire.org -p 2220
cat /etc/bandit_pass/bandit14
```

---

## level 14 → 15: talking to a raw tcp port

**concepts**

| concept | explanation | why it matters here |
|---|---|---|
| localhost | loopback address, always refers to the current machine | the service on port 30000 runs on the same bandit box you're already inside, nothing external to reach |
| port | a numbered endpoint a specific service listens on | 30000 is this level's custom service, unrelated to ssh's 2220 |
| `nc` (netcat) | general purpose tool for reading/writing raw bytes over a network connection | lets you manually push text at a port and read back the response, no protocol wrapper needed |

**solve**

```bash
echo "$(cat /etc/bandit_pass/bandit14)"
#copy the thing
nc localhost 30000
#enter the thing
#enter one more time
```

---

## level 15 → 16: same thing, over tls

**concepts**

| concept | explanation | why it matters here |
|---|---|---|
| tls wrapped service | some ports expect an encrypted handshake before accepting any data | plain `nc` will just hang or return nothing sensible against a tls only port |
| `openssl s_client` | generic tls client, negotiates the handshake first, then behaves like a raw connection | the right tool once you know a service expects encryption |
| self signed cert warning | openssl flags the connection as using an untrusted, self issued certificate | expected and safe to ignore here, bandit isn't using a real ca signed cert |

**solve**

```bash
echo "$(cat /etc/bandit_pass/bandit15)"
#copy
ncat localhost 30001
#paste the thing in
```

---

## level 16 → 17: scanning a port range for the real service

**concepts**

| tool/flag | meaning | why used here |
|---|---|---|
| `nmap` | dedicated port scanner | probes a whole range and reports what's open in one pass instead of testing 1000 ports by hand |
| `-p <range>` | restricts the scan to specific ports | `31000-32000` matches exactly what the level specifies |
| distinguishing plaintext vs tls | not every open port speaks the same protocol | only one port in the range actually hands back credentials, the rest just echo whatever you send |

is nmap's service detection guaranteed? not perfectly, it makes educated guesses off banners and behavior. verifying manually with `openssl s_client` (and falling back to `nc` if that fails) is still necessary.

**solve**

```bash
nmap -p 31000-32000 localhost
```

for each open port, try this:

```bash
ncat localhost 30001
```

once you find the one that responds sensibly, feed it the current password.

---

## level 17 → 18: diffing two files

**concepts**

| concept | explanation | why it matters here |
|---|---|---|
| `diff -u file1 file2` | compares two files, outputs the lines that differ (add `-u` for github style highlighting) | algorithmically finds the one changed line instead of you eyeballing two large files |
| `<` marker | line only present in file1 (the one listed first) | tells you the old version of the changed line |
| `>` marker | line only present in file2 (the one listed second) | tells you the new version, your actual target |

**solve**

```bash
diff -u passwords.old passwords.new
```

the `>` line is the answer, the `<` line right above it is just the old value for context.

---

## level 18 → 19: a `.bashrc` that logs you out

**concepts**

| concept | explanation | why it matters here |
|---|---|---|
| `.bashrc` | a per-user config script sourced automatically on every interactive bash session | it's been sabotaged with a logout command, killing your session before you get a usable prompt |
| `ssh user@host command` | runs one specific command remotely and returns, without a full interactive shell | this is standard, documented ssh behavior, not a hack |
| interactive vs non-interactive shell | interactive gives you a live prompt, non-interactive just runs one command and exits | `.bashrc` only fires for interactive sessions, that gap is exactly what you exploit |

**solve**

```bash
ssh bandit18@bandit.labs.overthewire.org -p 2220 "cat readme"
```

a normal interactive login attempt logs you out instantly, that's expected given the level's setup, not something broken on your end.

---

## level 19 → 20: your first setuid binary

**concepts**

| concept | explanation | why it matters here |
|---|---|---|
| setuid bit | a permission bit that makes a binary run with the file owner's privileges, not the caller's. no matter who you are, if the file is setuid'ed you run it as whoever created it. incredibly dangerous and infamous, especially for anything owned by root | the entire mechanism of the level, the binary is owned by bandit20 |
| `s` in permission bits | shows up where you'd normally see `x` in the owner's execute slot | visual confirmation the setuid bit is set |
| running with no arguments | well designed clis print usage instructions when called blind | the level tells you to do exactly this to learn the expected input |

is setuid a real os mechanism or just convention? real, an actual permission bit enforced by the kernel at execution time, not a naming trick.

**solve**

```bash
ls -la
./bandit20-do
./bandit20-do cat /etc/bandit_pass/bandit20
```

trying to `cat` that path directly as bandit19 fails with permission denied, you have to go through the setuid binary.

---

## level 20 → 21: setuid binary that connects back to you

**concepts**

| concept | explanation | why it matters here |
|---|---|---|
| setuid binary as network client | the binary connects out to a port you specify, reads a line, compares it to the current password | you supply both ends: a listener serving the password, and the binary connecting to it |
| two terminal workflow | one terminal hosts the listener, the other triggers the binary | lets you watch both sides of the exchange live |
| `nc -lp <port>` | `-l` = listen, `-p` = port, opens a listening socket | this is your side of the connection, serving the current password to whoever connects |
| `&` at the end | runs the command in background mode, giving you back the prompt | you have to act as both the host and the client here |
| `fg (x)` | x is the job number you want to bring back to the foreground | you have to actually watch/write to it interactively |

**solve**

```bash
# terminal 1
nc -lp 4444 < /etc/bandit_pass/bandit20 & #(run and put it to background)
./suconnect 4444 &
fg 1
#enter the thingie
```

---

## level 21 → 22: reading a cron job

**concepts**

| concept | explanation | why it matters here |
|---|---|---|
| cron | a time based job scheduler daemon running commands on a schedule | something on this box runs periodically and touches this level's password |
| `/etc/cron.d/` | directory holding system wide cron job definitions | the level explicitly points here for the config |
| reading a script before running it | tracing what a script actually does, line by line | the leak is usually a predictable output path the script writes to |

**solve**

```bash
ls -la /etc/cron.d/
cat /etc/cron.d/cronjob_bandit22
cat /usr/bin/cronjob_bandit22.sh
```

the script writes bandit22's password to a predictable `/tmp` path and chmods it world readable right before doing so, that's the leak.

```bash
cat /tmp/<filename the script wrote to>
```

---

## level 22 → 23: cron job with a derived filename

**concepts**

| concept | explanation | why it matters here |
|---|---|---|
| computed output path | the script builds its destination filename from a formula instead of hardcoding it | you don't need to catch it mid run, you can just compute the same value yourself |
| `md5sum` | computes an md5 hash of given input | the script hashes a fixed string plus `whoami`'s output to build the filename |
| `man 5 crontab` | man pages are split into numbered sections, section 5 covers file formats | disambiguates the crontab file syntax docs from the crontab command's own man page |

**solve**

```bash
cat /etc/cron.d/cronjob_bandit23
cat /usr/bin/cronjob_bandit23.sh
```

match the exact string the script builds, usually a fixed phrase concatenated with `$myname`:

```bash
echo "I am user bandit23" | md5sum
cat /tmp/<resulting md5 hash>
```

---

## level 23 → 24: planting your own script in a cron job

**concepts**

| concept | explanation | why it matters here |
|---|---|---|
| world writable execution directory | a directory anyone can write into, that a privileged process later executes files from | the actual vulnerability, privilege depends on who runs the file, not who wrote it |
| `chmod +x` | adds the execute bit to a file | your script needs this before the cron job's invocation logic will run it as a program |
| shebang line | the first line of a script telling the os which interpreter to use | without it the script may fail to run properly depending on how cron invokes it |
| self cleanup timing | the cron job deletes files right after running them | write your output somewhere persistent before your script vanishes, don't rely on inspecting it afterward |

**solve**

```bash
cat /etc/cron.d/cronjob_bandit24
cat /usr/bin/cronjob_bandit24.sh
ls -la /var/spool/bandit24
```

confirm it's writable, then drop a script that copies the target password somewhere persistent:

```bash
mkdir /tmp/$(whoami)-drop
cat > /tmp/$(whoami)-drop/grab.sh << 'EOF'
#!/bin/bash
cat /etc/bandit_pass/bandit24 > /tmp/$(whoami)-drop/out.txt
EOF
chmod +x /tmp/$(whoami)-drop/grab.sh
cp /tmp/$(whoami)-drop/grab.sh /var/spool/bandit24/
```

wait roughly a minute for the next cron tick, then:

```bash
cat /tmp/$(whoami)-drop/out.txt
```

---

## level 24 → 25: brute forcing a 4 digit pin

**concepts**

| concept | explanation | why it matters here |
|---|---|---|
| brute force | trying every possible value in a keyspace until one works | the only viable approach here, the pin isn't derivable or stored anywhere readable |
| keyspace size | total number of possible values, 10^4 = 10000 here | small enough for a computer to exhaust in seconds, this is why brute force is the intended path |
| for loop | run something a fixed number of times, here a c-like for loop over 0-10000 | need it to iterate through every combination |
| `printf` | formatted printing | `%04d` pads the number to exactly 4 digits, so you get 0000, 0001, 0002... instead of 0, 1, 2 |
| `\| ncat` | piping | whatever the loop outputs goes straight into ncat, no need for a separate script |

**solve**

a small script handling one persistent socket, since a clean pure bash version gets messy fast:

```bash
#!/bin/bash

password_from_previous="xxxx"  # insert it here

for ((i=0; i<10000; i++)); do
    printf "%s%04d\n" "$password_from_previous" "$i"
done | nc localhost 30002
```

---

## level 25 → 26: escaping a pager based restricted shell

**concepts**

| concept | explanation | why it matters here |
|---|---|---|
| `/etc/passwd` shell field | the last field in a passwd entry specifies the login shell binary | tells you exactly what program you land in on login, could be anything, not always `/bin/bash` |
| restricted/non-shell programs as a shell | any executable can technically be set as a login shell | bandit26's shell is a script that just displays a file and exits |
| pager escape features | tools like `more`/`less`/`vi` have built in ways to spawn a subshell from within them | if the fake shell turns out to be a pager, its own features become the escape hatch |
| terminal resize trick | pagers only paginate (rather than dump and exit) if the content doesn't fit the visible screen | shrinking the terminal forces `more` to actually stop and wait, which unlocks its interactive commands |

**solve**

```bash
cat /etc/passwd | grep bandit26
```

shows the shell field pointing at a custom script instead of bash.

```bash
cat /usr/bin/<that custom shell script>
```

reveals it runs `more` on a text file and exits. shrink your terminal window (fewer visible rows) before or during connecting so `more` is forced to paginate instead of dumping everything and exiting immediately:

```bash
ssh -i bandit26.sshkey bandit26@localhost
# once paginating, press v to open vi on the currently displayed content
```

inside vi, you're one step from a real shell:

```
:set shell=/bin/bash
:shell
```

---

## level 26 → 27: using the setuid binary you land next to

**concepts**

| concept | explanation | why it matters here |
|---|---|---|
| inherited privilege from an escape | whatever user context was active at the moment you escaped carries into your new shell | determines what you can actually reach next, having a shell isn't the same as having access |
| setuid recap | same mechanism as level 19/20 | worth checking for another setuid binary immediately after landing in a new shell |

**solve**

```bash
whoami
ls -la
```

a setuid binary owned by bandit27 sits in the home directory, same pattern as level 19/20.

```bash
./bandit27-do cat /etc/bandit_pass/bandit27
```

---

## level 27 → 28: password sitting in a git repo

**concepts**

| concept | explanation | why it matters here |
|---|---|---|
| `ssh://user@host/path` url | tells git to fetch over ssh instead of https or a local path | this repo is only reachable via ssh |
| `git clone <url>` | downloads a full copy of a remote repo, all files plus full history | your one command to pull everything down |
| working in `/tmp` | a writable scratch space, home directories are often locked down for writes in these levels | avoids permission headaches cloning into your home dir |

**solve**

```bash
mkdir whatever
git clone ssh://bandit27-git@localhost/home/bandit27-git/repo
cd repo
cat README
```

---

## level 28 → 29: password scrubbed from HEAD, alive in history

**concepts**

| flag/command | meaning | why used here |
|---|---|---|
| `git log` | lists commits newest first, with hash/author/date/message | your starting point, commit messages here are basically confessions |
| `git cat-file -p` | `-p` = "patch", shows the content | reveals exactly what content there was instead of just a hash |
| append only history | git never overwrites history, "removing" a line is just a new commit where it's absent | the old commit object with the real value is still fully addressable |

**solve**

```bash
git clone ssh://bandit28-git@localhost/home/bandit28-git/repo
cd repo
cat README.md
git log --oneline --graph --all
git cat-file -p #the earlier commit
```

---

## level 29 → 30: password on a different branch

**concepts**

| flag/command | meaning | why used here |
|---|---|---|
| `git branch -a` | `-a` = "all", lists every branch, local and remote-tracking | plain `git branch` might only show local branches, missing the one you actually need |
| `git checkout <branch>` | swaps your working directory to reflect a different branch's state | needed to actually view files as they exist on that branch |
| readme hints | in-repo flavor text nudging you toward a non-default branch | practical clue, not a mechanism, worth reading carefully |

why doesn't `git log` on master show commits from another branch? it only walks history reachable from your current branch. a separate branch can carry entirely different commits that never merged in.

**solve**

```bash
git clone ssh://bandit29-git@localhost/home/bandit29-git/repo
cd repo
cat README.md
git branch -a
git checkout dev
cat README.md
```

---

## level 30 → 31: password in an annotated tag

**concepts**

| concept/command | meaning | why it matters here |
|---|---|---|
| `git tag` | lists all tags in the repo | neither `git log` nor `git branch -a` will ever surface these, a distinct reference type |
| annotated vs lightweight tags | annotated tags store a full object with a message, like a mini commit, lightweight tags are just a bare pointer | this level's password sits in an annotated tag's message |
| `git show <tagname>` | for an annotated tag, prints the tag's message plus whatever commit it points to | how you actually read where the password lives |

**solve**

```bash
git clone ssh://bandit30-git@localhost/home/bandit30-git/repo
cd repo
cat README.md
git log
git branch -a
```

all dead ends deliberately, the password moved to a third reference type.

```bash
git tag
git show <tag name>
```

---

## level 31 → 32: force pushing past `.gitignore`

**concepts**

| flag/command | meaning | why used here |
|---|---|---|
| `.gitignore` | lists patterns git should skip by default during commands like `git add .` | the repo ships with `*.txt` in it, specifically blocking the file you're required to push |
| `git add -f <file>` | `-f` = "force", stages a file even if it matches a `.gitignore` pattern | the direct override, bypasses the ignore rule for just this one file |
| `git commit -m "<msg>"` | commits with an inline message | standard one-line commit |
| `git push origin master` | uploads local commits to the remote's master branch | without this the server never even sees your file |
| server side validation hook | the remote checks the pushed content against what's expected | not something you control, decides whether you get the next password back |

is `.gitignore` a security barrier? no, it's a convenience default for `git add .`, not access control. git happily accepts an explicitly forced add regardless.

**solve**

```bash
git clone ssh://bandit31-git@localhost/home/bandit31-git/repo
cd repo
cat README.md
cat .gitignore
```

```bash
echo "May I come in?" > key.txt
git add key.txt
```

git will actually warn you here that the file is ignored, worth reading instead of assuming the command silently failed.

```bash
git add -f key.txt
git commit -m "add key.txt"
git push origin master
```

the remote's response prints the next password directly if the filename and content match exactly what the readme specified.

---

## level 32 → 33: escaping an uppercase only shell

**concepts**

| concept | explanation | why it matters here |
|---|---|---|
| wrapper shell | a custom program set as your login shell that transforms input before running it | not a real interactive shell, just a filter sitting in front of one |
| `$0` | a positional parameter holding the path of the currently running shell/script | expanded by the shell into a literal value before the wrapper's transform logic can meaningfully touch it |
| why uppercasing `$0` doesn't break it | the wrapper mangles typed characters, `$0` is a variable expansion resolved by the shell itself | the actual gap being exploited, not a typed command getting mangled, but a variable expansion the wrapper never accounted for |
| setuid recap | the wrapper binary is itself setuid to the next user | escaping into a raw shell from inside it inherits that elevated identity |

**solve**

```
ls
```

fails instantly, `sh: 1: LS: not found`, confirming the wrapper uppercases everything you type before running it.

```
$0
```

drops you into a real, unmangled shell.

```bash
whoami
cat /etc/bandit_pass/bandit33
```

---

## level 33: the end

no puzzle left, just confirming the last password actually logs in.

```bash
ssh bandit33@bandit.labs.overthewire.org -p 2220
cat README.txt
```

that's the wargame. every level here builds on something earlier: the `./` and `--` trick from level 1 shows up again in level 4's decoy filenames, the setuid lesson from level 19/20 gets reused twice more, and the git levels (27-31) are really one lesson (clone) taught four different ways: history, branches, tags, and finally the inverse, writing instead of reading. worth doing once slowly rather than speedrunning with someone else's answers.

---

credit: overthewire bandit wargame — https://overthewire.org/wargames/bandit/

no passwords, keys, or flags are included anywhere in this writeup, per their rules.
