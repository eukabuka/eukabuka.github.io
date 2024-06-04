# Reading through vsftpd source

Recently I've been reading through vsftpd source(written by Chris Evans), and it has been
an educational experience. Here I'm planning on jotting down notes from the design
along with excerpts from the source.
* [Design doc](https://security.appspot.com/vsftpd/DESIGN.txt)
* [Trust relationships](https://security.appspot.com/vsftpd/TRUST.txt)
* [Changelog](https://security.appspot.com/vsftpd/Changelog.txt)

The following sections are from the documentation, and I've tried to attach the
corresponding source snippets to these sections.

### Creating a string abstraction
A custom string type is defined as:
```C
struct mystr
{
    char *p_buf;
    unsigned int len;
    unsigned int alloc_bytes;
};
```
The functions that operate on `mystr` can be grouped into the following categories:
* Allocation functions that take lengths of different types `char`, `unsigned
long`, `filesize_t`.
* Allocation-adjacent functions such as `str_[copy|dup|empty|free|trunc|reserve]`.
* Getter functions such as `str_[isempty|getlen|getbuf]`.
* Comparison functions such as `str_[strcmp|empty|equal_text]`.
* Modifer functions that modify the underlying content such as `str_append_*` and
`str_[upper|rpad|lpad|replace_*]`.
* Split functions TODO
* Find functions such as `str_locate_*`.
* Partition functions such as `str_[left|right|mid_to_end]` TODO
* 

All calls to libc functions go via an intermediary named `vsf_sysutil_*` that performs
sanity checks. sysutil functions are wrappers around libc functions with strict sanity
checks on the arguments(the process does some cleanup and exits if the arguments are not
sane).

### Separating out processes
vsftpd is split into privileged and unprivileged parts. These parts communicate over
socketpair. The privileged parts are minimal, and it exercises distrust of unpriv. The
privileged process creates a socket at port 20 for data, and sends it over to unpriv.

`main()` calls `vsf_standalone_main()` that does a bunch of initialization including
a listen and bind; and accept in a loop. The client sock in the child handling
the connection has stdin/stdout/stderr set to the sock.

`main()` calls `vsf_two_process_start()`, an interesting function.

The comms channel between priv parent and no-priv child is set up in `priv_sock_init`.
It essentially creates a unix sockerpair and sets that for use in the session.

Depending on whether the `isolate_network` flag is set, a `clone(CLONE_NEWNET)` or
`fork()` is called. The parent process then processes the login request.

`drop_all_privs()` then drops the privileges by calling `vsf_secutil_change_credentials`.
It then does the following:
1. It is possible to specify a config option `nopriv_user`. If this is set, the
corresponding password file entries are fetched.
2. It then drops all supplementary groups with `setgroups(0, NULL)`.
3. In twoprocess mode, it does not attempt to drop capabilities.


### chroot
If the unpriv process is compromised, its file system access will be constrained using
a chroot. Unpriv process does OpenSSL protocol parsing in both preauth and postauth
stages.

### capabilities
* The priv process dynamically calculates what privileges it needs. If no privileges are
needed, the priv process exits. As mentioned above this is only used in oneprocess mode
(disabled by default, and not recommended for general use).

### not relying on any external binaries
A minimal subset of `/bin/ls` functionality is implemented instead of trusting the
binary.

### not relying on complex library APIs
Functions that interact with the network, parse user data are considered dangerous. Some
examples given are `fnmatch()`(which may be used in FTP to list files with glob patterns)
and `gethostbyaddr()`(to resolve a host address).
All functions that deal with external libraries are wrapped around in `sysutil.c` and
`sysdeputil.c`.

Since string handling is a common source of memory errors, create a string type that
abstracts the underlying data and operate on it using a carefully audited set of APIs.

### ptrace sandbox, seccomp sandbox
sandbox is initialized in `vsf_two_process_start()`.
1. A global array `s_syscalls`(size tracked with `s_syscall_index`) is a list of syscalls
that are to be enabled. Syscalls can be allowed by calling `allow_nr()`. Syscalls that
are explicitely rejected can be recorded with `reject_nr`(which takes a syscall number
along with a errcode) #TODO.
2. `seccomp_sandbox_setup_prelogin()` and `seccomp_sandbox_setup_base()` allow 6-13
syscalls(depending on the config) and 11 syscalls(read, write, mmap, munmap, mremap,
gettimeofday, rt_sigreturn, restart_syscall, close, exit_group) respectively.
`(mremap, ENOSYS)` and `(socket, EACCES)` are rejected explicitely.
3. `seccomp_sandbox_lockdown()` then creates the `prog` PENDING
4. `seccomp_sandbox_setup_postlogin` PENDING

