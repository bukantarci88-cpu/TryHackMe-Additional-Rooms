# PHP2Shell

### Task 1 - Introduction

On 21 October 2026 the PHP project disclosed two vulnerabilities that, between them, let an unauthenticated attacker run commands on almost any PHP server with nothing but ordinary HTTP requests, and began shipping emergency fixes across every supported branch. No login, no file upload the application accepts, and no bug in the application itself: the flaw lives in the PHP runtime, below whatever framework or CMS happens to be installed. The disclosure was named **PHP2Shell**, and because the vulnerable code has been present since PHP 5.4.0 in 2012, it reaches fourteen years of releases at once.

The two issues are tracked as **CVE-2026-109618** and **CVE-2026-109619**, both scored **9.2 Critical** on CVSS v4.0. They were found by researchers at Wiz, OpenAI and Calif, reported privately to the PHP security team, and patched in PHP 8.5.12, 8.4.27, 8.3.36 and 8.2.35. Details leaked a day before the fix, and network sensors recorded exploitation attempts within hours of disclosure, some even before the patched releases were out, so this is a live threat rather than a theoretical one.

### Task 2 - A Harmless Feature: Upload Progress

The vulnerability exploits a legitimate, useful feature; understanding that underlying feature is essential to explaining how the attack works.

When a browser uploads a large file, the server has no simple way to tell the page how far the upload has got. PHP solves this with a feature called `session.upload_progress`. While PHP is still reading a `multipart/form-data` request body off the wire, it writes a running description of the upload, its name, its size and how many bytes have arrived so far, into the user's `$_SESSION`. A second request can then read that session data and report progress back to the browser. The feature is enabled by default, and it is switched on for a given upload by including a form field whose name matches the `session.upload_progress.name` setting, which defaults to `PHP_SESSION_UPLOAD_PROGRESS`.

[A multipart POST carrying the PHP_SESSION_UPLOAD_PROGRESS field causes PHP to write progress data into the session during body parsing, before request initialisation and before the target script runs.](https://cdn-images.tryhackme.com/room-content/645b19f5d5848d004ab9c9e2-1792662300601)

PHP parses request bodies and writes upload progress updates very early, long before environment initialization or any application code runs. This explains why application-level defenses (like in-process WAFs) cannot catch or block the attack. That has a consequence the feature's designers never intended to expose: reaching this code needs no `session_start()` in the target page, no valid login, and in fact no application logic at all. Any URL on the server that accepts a POST request is enough, even one whose script does nothing but print a single line. The pre-execution phase serves as the foothold for PHP2Shell, driven entirely by a standard multipart upload before any application code runs.

**Answer the questions below**

> What is the default name of the form field that switches on PHP's upload-progress tracking?
> 

PHP_SESSION_UPLOAD_PROGRESS

It matches the default value of PHP's `session.upload_progress.name` configuration setting.

### Task 3 - The Bug: A Bailout at the Wrong Moment

The upload-progress feature is not itself the vulnerability. The vulnerability is what happens when PHP runs out of memory while that feature is in the middle of its work, and to see it you need to know how PHP handles a fatal error.

PHP caps how much memory a single request may use with the `memory_limit` setting. When a request tries to exceed it, PHP cannot simply carry on, so it performs what the engine calls a **bailout**: it uses a `longjmp()` to jump straight back to a safe recovery point, abandoning whatever it was doing. This is the mechanism behind the familiar "Allowed memory size exhausted" fatal error. Normally a request tidies up after itself, freeing memory and clearing the pointers that referred to it, but a bailout skips all of that. It is an emergency exit, not an orderly shutdown.

The flaw is a matter of timing. If the out-of-memory condition strikes *after* the engine has freed a block of memory but *before* it has cleared the pointer that still refers to that block, the bailout leaves the pointer behind, pointing at memory that is no longer valid. This is a classic dangling pointer, classified as **CWE-824: Access of Uninitialised Pointer**.

[An out-of-memory bailout fires between freeing a block and clearing its pointer, so a dangling pointer survives the dead request and is reused by the next request on the same Apache worker.](https://cdn-images.tryhackme.com/room-content/645b19f5d5848d004ab9c9e2-1792662302867)

On its own, a dangling pointer in a request that has just died would be harmless, because the request is over. What makes it dangerous is the server model. Under Apache's **prefork** configuration with `mod_php`, a single worker process serves one request after another without restarting between them. The stale pointer therefore does not vanish with the failed request; it stays in the worker's memory, and the *next* request to land on that same worker finds the engine reaching through a pointer it never reset. An attacker who controls what gets allocated into that freed space in the second request controls what the pointer now points at.

From this single defect the researchers built two distinct attacks, which is why PHP2Shell carries two CVE numbers. **CVE-2026-109618** is a *memory-disclosure* flaw: the stale pointer is read back as a string with an attacker-influenced, oversized length, and PHP returns the result in a response header, handing the attacker a window into the worker's memory. **CVE-2026-109619** is the *remote-code-execution* flaw, which turns the corrupted state into control over the program's flow. The PHP changelog addresses both with a single description, "Avoid state corruption via bailout", which names the root cause precisely. The next three tasks follow that chain in order: confirm the target, use the disclosure to defeat ASLR, and then turn the corruption into a shell.

**Answer the questions below**

> What one-word name does the PHP engine give to the longjmp it performs to abort a request that hits a fatal error such as running out of memory?
> 

bailout

A **bailout** is an emergency abort mechanism. When a request encounters an unrecoverable fatal error such as exceeding the `memory_limit,`the engine uses `longjmp()` to immediately jump to a safe recovery point. It stops whatever it is doing, bypasses normal function returns, and skips the standard, orderly cleanup process.

> What is the CWE identifier for the weakness class behind PHP2Shell? (format: CWE-NNN)
> 

CWE-824

CWE-824 (Access of Uninitialized Pointer) applies to PHP2Shell because of how the PHP engine handles memory during emergency out-of-memory bailouts.

### Task 4 - Reconnaissance: Confirming the Target

The vulnerable server runs as a container. List it and read the PHP version it ships:

```bash
ubuntu@tryhackme:~$ docker ps --format '{{.Names}}  {{.Image}}  {{.Status}}'
php2shell-target  php:8.4.26-apache  Up 3 minutesubuntu@tryhackme:~$ docker exec php2shell-target php -v
PHP 8.4.26 (cli) (built: Oct 21 2026 00:31:19) (NTS)
Copyright (c) The PHP Group
Zend Engine v4.4.26, Copyright (c) Zend Technologies
```

The server is PHP **8.4.26**, one release short of the 8.4.27 fix, so it is vulnerable. Now look at the page you will be attacking:

```bash
ubuntu@tryhackme:~$ curl -s http://127.0.0.1:8088/index.php
OKubuntu@tryhackme:~$ docker exec php2shell-target cat /var/www/html/index.php
<?php echo "OK\n"; ?>
```

The entire application is one line that prints `OK`. Because the exploit targets the PHP runtime rather than the application itself, even a blank, bug-free page is vulnerable, proving that secure application code offers no protection against PHP2Shell.

The server runs the ordinary prefork `mod_php` setup with `session.upload_progress` enabled as it is by default. For the lab it is cut down to a single worker, and `memory_limit` is set low, so the dangling pointer is reliably reused and the out-of-memory condition is easy to provoke; a production host would run many workers with a higher limit, which makes the timing harder to hit but not impossible. The exploit you will use is already on the machine:

```bash
ubuntu@tryhackme:~$ ls ~/php2shell-bypass-aslr-exp
docker  exploit.py  out  README.md  readme_en.md  src  tools
```

With the target identified and confirmed vulnerable, you can move on to the first half of the attack: making the server hand you its own memory.

**Answer the questions below**

> What version of PHP is the target server running?
> 

8.4.26

docker exec php2shell-target php -v (It is a command to check the PHP version inside a running Docker container.)

> What single word does the target page at /index.php print in response?
> 

OK

<img width="1035" height="325" alt="image" src="https://github.com/user-attachments/assets/a5306719-82db-4a45-ba31-3990f2508431" />

 

- curl command silently (-s) sends an HTTP request to the local web server running on port 8000 to fetch and display the contents of index.php.
- the output "OK" means that the local web server successfully processed the request and sent back a response. it indicates: HTTP 200 Success and The target script (index.php) simply returned the plain text string "OK" as its entire body.

### Task 5 - Leaking Memory and Defeating ASLR (CVE-2026-109618)

CVE-2026-109618 bypasses ASLR (Address Space Layout Randomisation) protection by exploiting the dangling pointer to leak the worker process's memory address layout.

An out-of-memory error leaves a stale pointer to a session ID, which a subsequent request reuses, allowing the attacker to inflate the string's length far beyond its actual data. When PHP sends the session cookie, it reads that oversized session-id string and returns it to the client in the **Set-Cookie: PHPSESSID** response header. The header now carries whatever happened to sit in the worker's memory after the string, which the research measured at up to roughly 67 KB of process memory per request. In a real deployment that memory can hold other users' passwords, session cookies, API keys, and even TLS private keys; for the exploit, it holds the addresses it needs.

[A stale PHPSESSID pointer is read with an oversized length, so PHP copies adjacent worker memory into the Set-Cookie response header, disclosing heap and library addresses to the attacker.](https://cdn-images.tryhackme.com/room-content/645b19f5d5848d004ab9c9e2-1792662305210)

The exploit performs two successive memory leaks, first exposing the heap base address and then the `libphp` library base address, to completely defeat ASLR by mapping out the worker's exact memory layout before proceeding to code execution.

```bash
ubuntu@tryhackme:~$ cd ~/php2shell-bypass-aslr-exp
ubuntu@tryhackme:~/php2shell-bypass-aslr-exp$ python3 exploit.py -u http://127.0.0.1:8088/index.php
[*] target http://127.0.0.1:8088/index.php (cmd='id>/out/p1')
[*] offsets loaded from out/offsets.env
[*] rotating worker over HTTP
[*] stage 1: heap leak (attempt 1/3)
[+] chunk base = 0x70d1a2c00000
[*] rotating worker over HTTP
[*] stage 2: libphp leak (attempt 1/3)
[+] libphp base = 0x70d1a4911000
```

The two `[+]` lines are the disclosure: the heap chunk base and the `libphp` base, each read straight out of a `Set-Cookie` header. The addresses differ on every run, which is ASLR doing its job; the attack simply reads them out each time rather than guessing them. With the heap and the library located, the second CVE can take over and turn the corruption into execution.

**Answer the questions below**

> Which HTTP response header does CVE-2026-109618 abuse to return leaked process memory to the attacker?
> 

**Set-Cookie: PHPSESSID** (It is an HTTP header used by PHP to assign a unique session ID cookie to a user.)

 

<img width="954" height="229" alt="image 1" src="https://github.com/user-attachments/assets/42053e4d-33ae-4b81-9baa-514706b745fb" />


the successful execution of the Python exploit script for PHP2Shell, which performs the heap and `libphp` memory leaks to bypass ASLR and achieves remote code execution (`id`), displaying `www-data` user privileges.  

### Task 6 - From a Leak to a Shell (CVE-2026-109619)

CVE-2026-109619 leverages the memory addresses discovered by the leak to place a forged object into freed memory, hijacking PHP's clean-up mechanisms to achieve remote code execution.

When PHP destroys a value, it may call a function pointer stored alongside it, a destructor, to release the value's resources. The exploit reoccupies the dangling reference with a forged internal structure (a `HashTable`) whose destructor pointer, `pDestructor`, has been set to the address of `system()` inside the PHP library, an address the leak in the previous task made known. When the engine tidies up that value at the end of request handling, it believes it is calling a harmless destructor and instead calls `system()` with attacker-controlled bytes as its argument. The command runs as the web server user. Nothing in the attack needs the C library or any gadget outside PHP itself; the whole finish is built from the engine's own code.

[The full chain: rotate the worker, leak the heap base, leak the libphp base to defeat ASLR, then forge a HashTable whose destructor points at system() so cleanup executes the attacker's command as the web user.](https://cdn-images.tryhackme.com/room-content/645b19f5d5848d004ab9c9e2-1792662307387)

Running the exploit with no command argument performs the whole chain and proves execution by running `id` on the target:

```bash
ubuntu@tryhackme:~/php2shell-bypass-aslr-exp$ python3 exploit.py -u http://127.0.0.1:8088/index.php
[*] target http://127.0.0.1:8088/index.php (cmd='id>/out/p1')
[*] offsets loaded from out/offsets.env
[*] rotating worker over HTTP
[*] stage 1: heap leak (attempt 1/3)
[+] chunk base = 0x70d1a2c00000
[*] rotating worker over HTTP
[*] stage 2: libphp leak (attempt 1/3)
[+] libphp base = 0x70d1a4911000
[*] rotating worker over HTTP
[*] stage 3: HTDATA=0x70d1a2c690d8 ZIF=0x70d1a4d76370
[+] W3 attempt 1
PROOF: uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

That is remote code execution, from nothing but multipart POST requests to a page that prints `OK`, against a server with no application vulnerability of any kind. The whole chain completes in under ten seconds.

To retrieve the flag, run a command of your own. The payload that carries it is tiny, with only fourteen bytes for the command and its redirection, so the flag is exposed at the short path `/f` on the target. The command below reads it into the shared `out` directory, which the tool then prints back:

```bash
ubuntu@tryhackme:~/php2shell-bypass-aslr-exp$ python3 exploit.py -u http://127.0.0.1:8088/index.php -c 'cat /f>/out/p1' --proof-file out/p1
[*] target http://127.0.0.1:8088/index.php (cmd='cat /f>/out/p1')
[*] offsets loaded from out/offsets.env
[*] rotating worker over HTTP
[*] stage 1: heap leak (attempt 1/3)
[+] chunk base = 0x70d1a2c00000
[*] rotating worker over HTTP
[*] stage 2: libphp leak (attempt 1/3)
[+] libphp base = 0x70d1a4911000
[*] rotating worker over HTTP
[*] stage 3: HTDATA=0x70d1a2c690d8 ZIF=0x70d1a4d76370
[+] W3 attempt 1
PROOF: THM{...}
```

**Answer the questions below**

> When the exploit runs, which system user do the commands execute as?
> 

www-data
 

<img width="909" height="229" alt="image 2" src="https://github.com/user-attachments/assets/1ed29671-7b89-4af0-8232-595f0b00ff62" />


> What is the flag returned by the exploit?
> 

THM{b4ilout_l3aves_th3_h3ap_dang1ing}

 

<img width="1075" height="246" alt="image 3" src="https://github.com/user-attachments/assets/caa549ed-64be-49cc-9953-46552d08b645" />


### Task 7 - Detection, Mitigation and the Real Fix

Because PHP2Shell exists in the PHP runtime rather than application code, fixing it requires updating the PHP package or container base image to a patched version (8.5.12, 8.4.27, 8.3.36, or 8.2.35) and restarting all workers, as standard application updates are ineffective. Until that patch can be applied, several interim measures reduce exposure, though the PHP team is clear that none is a complete substitute for patching. Disabling the feature the attack relies on, by setting `session.upload_progress.enabled = 0` and restarting all workers, removes the trigger for most of the reported techniques. Turning off the opcode cache with `opcache.enable = 0` frustrates the code-execution path. On PHP 8.5, `fatal_error_backtraces = 0` addresses CVE-2026-109619 specifically. At the network edge, rejecting `multipart/form-data` requests blocks the delivery mechanism, at the cost of breaking legitimate file uploads. Running the PHP process with least privilege limits what an attacker gains if the chain does land. That highlights a crucial limitation: in-process Web Application Firewalls running as PHP modules are completely blind to PHP2Shell because the attack executes during request parsing, long before the application or the module ever receives control.

The attack can be detected by monitoring for key indicators in logs and traffic: abnormally large `PHPSESSID` values in `Set-Cookie` headers, large multipart POST requests with high field counts involving `PHP_SESSION_UPLOAD_PROGRESS`, and frequent PHP worker crashes or out-of-memory errors.

**Answer the questions below**

> Which PHP 8.4 release fixes PHP2Shell?
> 

8.4.27 (the PHP package or container base image to a patched version 8.5.12, 8.4.27, 8.3.36, or 8.2.35) 

> Which php.ini directive, set to 0, disables the feature the attack relies on for delivery?
> 

session.upload_progress.enabled (**`session.upload_progress.enabled`** is a PHP configuration directive (`php.ini`) that determines whether PHP tracks the progress of file uploads and stores that information in the user's session.)

### Task 8 - Conclusion

PHP2Shell is a reminder that the software stack does not stop at the application. A single mistake in how the PHP engine recovered from running out of memory, present and unnoticed since 2012, exposed an unauthenticated path to code execution on a large share of the web's servers at once, independent of which framework or CMS sat on top. The lesson for defenders is to treat the runtime as part of the attack surface and to patch it with the same urgency as the application, because a flaw this deep cannot be reached by securing the layers above it.

That captures the full mechanics of the PHP2Shell exploit chain—from triggering the OOM bailout and bypassing ASLR to hijacking PHP's cleanup mechanisms for remote code execution, all starting from a standard runtime feature like session upload progress. It serves as a striking demonstration of how runtime-level flaws bypass traditional application defenses entirely.
