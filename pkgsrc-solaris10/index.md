# Using pkgsrc on Solaris 10 SPARC in 2024

I am trying to collect all ideas/hacks I used to build software using pkgsrc on Solaris 10 SPARC.
The process is in progress. I am happy to get comments, ideas and exchange the experience: **mercurius(@)elming.org**

This page will be updated when I get new results.

 * [TL;DR](#tldr)
 * [Why Solaris 10?](#why-solaris-10)
 * [The process](#the-process)
 * [Binutils or if something is wrong](#binutils-or-if-something-is-wrong)
 * [Solaris 10 compat library](#solaris-10-compat-library)
    * [How to use it](#how-to-use-it)
 * [Other issues I faced](#other-issues-i-faced)
    * [devel/libuv](#devellibuv)
    * [misc/rhash](#miscrhash)
    * [devel/glib2](#develglib2)
    * [devel/cmake](#develcmake)
    * [graphics/gdk-pixbuf2](#graphicsgdk-pixbuf2)
    * [devel/pango](#develpango)
    * [graphics/netpbm](#graphicsnetpbm)
    * [security/gnutls](#securitygnutls)
    * [graphics/MesaLib](#graphicsmesalib)
 * [Getting new certificates](#getting-new-certificates)
 * [The result](#the-result)
 * [The system python](#the-system-python)
 * [GCC 9](#gcc-9)

## TL;DR
 
More or less, it’s successful. I am pleasantly surprised.  
There’s Python 3.12, OpenSSL 3.3.1 and a lot of modern software working on Solaris 10 SPARC!

I use gcc 5.5 from OpenCSW. I installed Sun Studio 12.3 but I wasn't successful in building anything in the pkgsrc tree, and most of the packages need gcc.
Also, I built gcc 9.5 from source, see [GCC 9](#gcc-9).

## Why Solaris 10?

I have Sun Ultra 60 and Sun Blade 100, Solaris 11 cannot be installed on them.  
OpenIndiana can, but it needs **zfs** and both of them don't have enough RAM.   
There's also another illumos-based system called [Tribblix](https://tribblix.org), it looks nice but I decided to try on the original **Sun Solaris 10 (01/13, the latest one)**.

I don't have access to original Oracle repositories for patches and updates. Then I have the system how it comes from the DVD.

## The process

pkgsrc HOWTO is here: [https://wiki.netbsd.org/pkgsrc/how_to_use_pkgsrc_on_solaris/](https://wiki.netbsd.org/pkgsrc/how_to_use_pkgsrc_on_solaris/)  
The official Solaris README: [https://cdn.netbsd.org/pub/pkgsrc/current/pkgsrc/bootstrap/README.Solaris](https://cdn.netbsd.org/pub/pkgsrc/current/pkgsrc/bootstrap/README.Solaris) - there's more information about Sun Studio, etc.  
There's a wiki page from 2007, it's old but can be useful: [https://wtf.hijacked.us/wiki/index.php/Pkgsrc_on_solaris](https://wtf.hijacked.us/wiki/index.php/Pkgsrc_on_solaris)

I took [pkgsrc 2024Q3](https://ftp.netbsd.org/pub/pkgsrc/pkgsrc-2024Q3/pkgsrc-2024Q3.tar.gz). The newer version has Perl 5.40, which I couldn't build on Solaris 10.  
(There was an issue in Github [https://github.com/Perl/perl5/issues/22728](https://github.com/Perl/perl5/issues/22728) – it’s closed, then maybe I'll give it a try again later.)

`sunfreeware.com` is not available anymore (at least, for free), then I am using **gcc5** and other utilities (mentioned in the HOWTO) from [https://www.opencsw.org/](https://www.opencsw.org/) 
Also, I had to install **sqlite3** from OpenCSW. Then, bootstrap and let's go!

(Just FYI, this is the full list of packages I installed using OpenCSW's `pkgutil`: [http://lizaurus.com/solaris10/pkgutil_list](http://lizaurus.com/solaris10/pkgutil_list). I think, not all of them are necessary anymore, when I have many pkgsrc's packages.

My mk.conf is here: [http://lizaurus.com/solaris10/pkgsrc-solaris10/bootstrap/mk.conf](http://lizaurus.com/solaris10/pkgsrc-solaris10/bootstrap/mk.conf)

## Binutils or if something is wrong

It’s tricky sometimes with binutils (like `as`, `ar` and especially `ld`).  
**Solaris SPARC has its own peculiar ABI** with 32-bit and 64-bit binaries (throw the manual to me to get more information!). `pkgsrc` has `ABI` option in `mk.conf` but sometimes I got an `ELFCLASS` error. It often helped to explicitly specify `CFLAGS+=-m64 CXXFLAGS+=-m64`.  
And I could not use GNU `ld` because Solaris `/usr/ccs/bin/ld` seems to know better about those things. I got errors about incompatible libraries. If I give it a try again, I'll share the experience.

But, often packages are successfully built with binutils which the process catches (usually, Solaris `/usr/ccs/bin` ones), but sometimes something is failing. Then:

- use environment variables like AR, AS and try another utility
- if the build process catches a wrong utility, create a symlink / adjust paths
- if there's an error, in most cases Google knows the answer or give you hints
- in some cases, check and add environment variables (`CFLAGS`, `LDFLAGS`). Sometimes I had to add explicitly `-m64` because of Solaris ABI
- in some other cases, libraries cannot be found, just add them to `LDFLAGS` explicitly in Makefile
- try to use `libsol10-compat` (see below)
- ...or you have to patch the code/the makefiles/meson build files... In some cases, the compiler suggested me which definitions you need to use.

## Solaris 10 compat library

Something is failing because Solaris is specific and doesn't have something which is available in other UNIX systems (constant definitions, header files, etc.).
And there's Solaris 10 compat library, big thanks to Pekdon for this: [https://github.com/pekdon/libsol10-compat/](https://github.com/pekdon/libsol10-compat/)

It has pkgsrc-compatible stuff to make a package... let's use it

```bash
export TMPDIR=/stuff/tmp # or another tmpdir/homedir/...
export PKGSRC=/stuff/pkgsrc # pkgsrc root directory, usually /usr/pkgsrc
cd $TMPDIR
git clone https://github.com/pekdon/libsol10-compat.git
cd libsol10-compat/pkgsrc
gcp -pruv libsol10-compat/ ${PKGSRC}/devel/ # I use GNU cp, Solaris cp -r will also work
cd ../..
```

then, the repo directory must to be renamed to libsol10-compat-0.1.0 and packed to the archive:

(we are in $TMPDIR)

```bash
mv libsol10-compat libsol10-compat-0.1.0
bsdtar cvzf ${PKGSRC}/distfiles/libsol10-compat-0.1.0.tar.gz libsol10-compat-0.1.0
cd ${PKGSRC}/devel/libsol10-compat/
bmake NO_CHECKSUM=yes
```

...bmake unpacks our archive, but then fails because we need to run `autoreconf -i` to have configure

```bash
cd ${PKGSRC}/devel/libsol10-compat/work/libsol10-compat-0.1.0
autoreconf -i
cd ../..
bmake NO_CHECKSUM=yes
bmake install
```

And yes! there's a package! Thank you again Pekdon!

**Newer version 0.2.x**: I had to add implementation of `open_memstream` to it, then there's a fork: [https://github.com/merclangrat/libsol10-compat](https://github.com/merclangrat/libsol10-compat). I guess there will be more functions added.

### How to use it

It installs `libsol10_compat_patch_pkgsrc` which patches original pkgsrc files and takes two arguments:
- the list of packages to be patched (there's "packages" file in the repo, but you can create your own,
- the root directory of pkgsrc tree, e.g. /usr/pkgsrc (I use /stuff/pkgsrc)

There's a blog post about trying Solaris 10 as a desktop: [https://pekdon.pekwm.se/posts/solaris_desktop/](https://pekdon.pekwm.se/posts/solaris_desktop/) - very useful!

## Other issues I faced

Of course this list isn't full, feel free to ask me if you have a specific issue. Sometimes I didn't document thoroughly what happened, sorry for that!

too many open files - Solaris has ulimit as usual, but it cannot be more than set in `/etc/system`. And if it's not set, the default is 256.
My settings are:

```bash
set rlim_fd_max=166384
set rlim_fd_cur=32768
```

after changing this file, you need to reboot!

### devel/libuv

 patched tcp.c but linking libnbcompat fails, add `-fPIC` flag [https://gnats.netbsd.org/56668](https://gnats.netbsd.org/56668)

### misc/rhash

take newer version (1.4.5) from the next pkgsrc tarball (2024Q3), it builds successfully

### devel/glib2

I used libsol10-compat, but it still thrown an error in 

```bash
[466/1428] Compiling C object glib/tests/getpwuid-preload.so.p/getpwuid-preload.c.o
FAILED: glib/tests/getpwuid-preload.so.p/getpwuid-preload.c.o 

./glib/tests/getpwuid-preload.c:71:22: error: conflicting types for 'getpwnam_r'
 DEFINE_WRAPPER (int, getpwnam_r, (const char     *name,
                      ^
../glib/tests/getpwuid-preload.c:35:15: note: in definition of macro 'DEFINE_WRAPPER'
   return_type func argument_list
               ^
In file included from ../glib/tests/getpwuid-preload.c:24:0:
/usr/include/pwd.h:139:12: note: previous declaration of 'getpwnam_r' was here
 extern int getpwnam_r(const char *, struct passwd *, char *,
            ^
../glib/tests/getpwuid-preload.c:82:22: error: conflicting types for 'getpwuid_r'
 DEFINE_WRAPPER (int, getpwuid_r, (uid_t           uid,
                      ^
../glib/tests/getpwuid-preload.c:35:15: note: in definition of macro 'DEFINE_WRAPPER'
   return_type func argument_list
               ^
In file included from ../glib/tests/getpwuid-preload.c:24:0:
/usr/include/pwd.h:138:12: note: previous declaration of 'getpwuid_r' was here
 extern int getpwuid_r(uid_t, struct passwd *, char *, int, struct passwd **);
```

This file just has different functions declarations, feel free to adjust them how they're declared in /usr/include/pwd.h and run bmake again.

### devel/cmake

cmake needs -lncurses because it tries to link itself with Solaris curses library + new gcc

I removed two files connected to GHS (they caused error about `gbuild` from Green Hills software, I guess we don't need that), also `bmake package SSP_SUPPORTED=no` (I think it can be solved by another way, for example symlinking `/usr/local/lib/sparcv9/libssp.so` to `/usr/pkg/lib`).

How long did it take to build cmake? Here it is:

```bash
real    786m29.379s
user    708m1.986s
sys     44m31.926s
```

it's built with new gcc and needs libs from the archive (see [GCC 9](#gcc-9) (gcc5 could't build it).

### graphics/gdk-pixbuf2

**This one is seriously broken. It builds but doesn't work. I am trying to find a way**.  
I had to disable tests completely in meson build after initial configure (stopped by ctrl+c, disabled and started `bmake` again).  
Also, it has an issue with `systeminfo.h` - just patch the file `pixops.c` to include it unconditionally.

### devel/pango

Building pango was tricky. There were linker problems, and because I was not successful to use `gld`, I had to replace `-z defs` to `-z nodefs` in meson build (also, after initial configure).
`pango` and `cairo` seem to work well because I could build `rxvt-unicode` and there's a nice terminal! Woohoo!

### graphics/netpbm

`netpbm` mysteriously has just `cc` without paths in `config.mk.in`, and it doesn't get changed by configure scripts. Putting `cc` symlink somewhere to `/usr/pkg/bin` didn't help, I had to change the path in the file and run bmake again.

### security/gnutls

Oh, that was the trickiest one! I don't know why building via `bmake` was failing, because if I used `gmake` in the source directory, it worked. I had to build it like this, but then use `readelf` and `elfedit` to fix binaries because there were relative paths.  
I think to try again and figure out what was happening there to make it buildable via `bmake` properly.

### graphics/MesaLib

It uses `static_assert` which is not available in Solaris, I patched the code and could build it.

Who else helped me? ChatGPT. sometimes it provides too much information, but usually I brought my errors to it and it could help where to look/patch/fix/etc.

## Getting new certificates

Even if `security/openssl` is installed, it cannot check certificates. There are older ones by OpenCSW in `/etc/opt/csw/ssl/certs`. I guess they can be used but we can install them using pkgsrc, too

I reminded that I had ca-certificates in Linux, but the package from pkgsrc needs `py-cryptography` which needs **Rust** for building, and Rust isn't available for Solaris-SPARC (yet).

But it's just necessary to install `security/mozilla-rootcerts-openssl` to get newer certificates. They will be stored in /usr/pkg/etc/openssl/certs, and then everything works well!

## The result

The packages I have already built are here: [http://lizaurus.com/solaris10/pkgsrc-solaris10/](http://lizaurus.com/solaris10/pkgsrc-solaris10/)
Almost all of them have been built by gcc5 from OpenCSW and some tools need its `libgcc_s`.

Packages built by gcc9 (see below) are in a subfolder (cmake, graphite, harfbuzz).

After the bootstrap, there're `pkg_*` utilities and those packages can be just installed without rebuilding. My bootstrap is also there: [http://lizaurus.com/solaris10/pkgsrc-solaris10/bootstrap](http://lizaurus.com/solaris10/pkgsrc-solaris10/bootstrap)

To make life simpler, I configure my PATH to look in /usr/pkg first:
`PATH=/usr/pkg/bin:/opt/csw/bin:/usr/sfw/bin:/usr/bin:`

and for root:
`PATH=/usr/pkg/sbin:/opt/csw/sbin:/usr/sfw/sbin:/usr/sbin:/usr/pkg/bin:/opt/csw/bin:/usr/sfw/bin:/usr/bin:`

## The system python

I think it's a good way to have `python3` and `python` symlinked to `/usr/pkg/bin/python3.12`. There's `/usr/sfw/bin/python` coming with Solaris, its version is 2.3.3. Quite old, eh?

## GCC 9

I wasn't able to build the package (`devel/gcc9`) using OpenCSW's gcc5, but I was able to build it from source. It took ca. **2 days** on my Sun Blade 100.

```bash
-bash-3.2# /usr/local/bin/gcc9.5 -v
Using built-in specs.
COLLECT_GCC=/usr/local/bin/gcc9.5
COLLECT_LTO_WRAPPER=/usr/local/libexec/gcc/sparcv9-sun-solaris2.10/9.5.0/lto-wrapper
Target: sparcv9-sun-solaris2.10
Configured with: /stuff/tmp/gcc-9.5.0/configure --program-suffix=9.5 --enable-languages=c,c++ -v --enable-obsolete --enable-ld=yes
Thread model: posix
gcc version 9.5.0 (GCC)
```

gcc 9.5 to be installed in `/usr/local` : [http://lizaurus.com/solaris10/gcc](http://lizaurus.com/solaris10/gcc) - just unpack the archive!

I used Solaris as (`/usr/ccs/bin/as`) and ld (`/usr/ccs/bin/ld`), also --enable-obsolete because gcc 9.5 is the last version supporting Solaris 10.

The next step is to try to build gcc9 in the pkgsrc using this gcc9! Then, we'll have a package which can be installed without any hacks, and re-build those packages which need newer gcc "by a proper way".

