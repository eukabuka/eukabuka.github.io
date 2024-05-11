# Reading through vsftpd source

Recently I've been reading through vsftpd source(written by Chris Evans), and it has been
an educational experience. Here I'm planning on jotting down excerpts from the design
along with excerpts from the source.
* [Design doc](https://security.appspot.com/vsftpd/DESIGN.txt)
* [Trust relationships](https://security.appspot.com/vsftpd/TRUST.txt)
* [Changelog](https://security.appspot.com/vsftpd/Changelog.txt)

The following is gathered from the above docs and the source code.

### Separating out processes
vsftpd is split into privileged and unprivileged parts. These parts communicate over
socketpair. The privileged parts are minimal, and it exercises distrust of unpriv. The
privileged process creates a socket at port 20 for data, and sends it over to unpriv.

### chroot
If the unpriv process is compromised, its file system access will be constrained using
a chroot. Unpriv process does OpenSSL protocol parsing in both preauth and postauth
stages.

### capabilities
* The priv process dynamically calculates what privileges it needs. If no privileges are
needed, the priv process exits.
* The priv process receives the child username and password.

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




