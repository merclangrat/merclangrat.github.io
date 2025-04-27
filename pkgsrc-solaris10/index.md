# Using pkgsrc on Solaris 10 SPARC in 2024/25

I am trying to collect all ideas/hacks I used to build software using pkgsrc on Solaris 10 SPARC.
The process is in progress. I am happy to get comments, ideas and exchange the experience: **mercurius(@)elming.org**

This page will be updated when I get new results.

 * [TL;DR](#tldr)
 * [Why Solaris 10?](#why-solaris-10)
 * [The process](#the-process)
 * [Solaris ABI](#solaris-abi)
 * [Binutils and versioned symbols](#binutils-and-versioned-symbols)
 * [too many open files](#too-many-open-files)
 * [If something is wrong](#if-something-is-wrong)
 * [Solaris 10 compat library](#solaris-10-compat-library)
    * [How to use it](#how-to-use-it)
 * [Getting new certificates](#getting-new-certificates)
 * [Adding Solaris fonts](#adding-solaris-fonts)
 * [The result](#the-result)
 * [The system python](#the-system-python)
 * [GCC 9](#gcc-9)
 * [Some issues related to specific packages](#some-issues-related-to-specific-packages)
    * [devel/libuv](#devellibuv)
    * [misc/rhash](#miscrhash)
    * [devel/glib2](#develglib2)
    * [devel/cmake](#develcmake)
    * [databases/shared-mime-info](#databasesshared-mime-info)
    * [graphics/gdk-pixbuf2](#graphicsgdk-pixbuf2)
    * [devel/pango](#develpango)
    * [graphics/netpbm](#graphicsnetpbm)
    * [security/gnutls](#securitygnutls)
    * [graphics/MesaLib](#graphicsmesalib)
    * [x11/gtk3](#x11gtk3)
    * [www/netsurf](#wwwnetsurf)
    * [textproc/groff](#textprocgroff)
    * [x11/fltk13](#x11fltk13)
    * [net/tigervnc](#nettigervnc)

## TL;DR
 
More or less, it’s successful. I am pleasantly surprised.  
There’s Python 3.12, OpenSSL 3.3.1, Ruby 3.x and a lot of modern software working on Solaris 10 SPARC!

I use gcc 5.5 from OpenCSW. I installed Sun Studio 12.3 but I wasn't successful in building anything in the pkgsrc tree, and most of the packages need gcc.
Also, I built gcc 9.5 from source, see [GCC 9](#gcc-9).

**Those packages are 64-bit. pkgsrc suggests to use ABI=64 in its manual (see below). I think I will try to build 32-bit ones too, or you can try if you want.**

I tried to collect something which I faced during building specific packages - [here is the list](packages.md).  
Of course this list isn't full, feel free to ask me if you have a specific issue. Sometimes I didn't document thoroughly what happened, sorry for that!

**Coming soon**: Emulators on Solaris 10 SPARC!

## Why Solaris 10?

I have Sun Ultra 60 and Sun Blade 100, Solaris 11 cannot be installed on them.  
OpenIndiana can, but it needs **zfs**. I purchased RAM for the Blade and it now has 1.5G... would be enough? I'll give it a try. Later.  
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

pkgsrc uses buildlink methodology [https://www.netbsd.org/docs/pkgsrc/buildlink.html](https://www.netbsd.org/docs/pkgsrc/buildlink.html) and symlinks necessary headers, libraries and pkgconfig `.pc` files into `work/.buildlink` subdirectory. It took some time for me to understand how that works! But yes, even if the library is installed but there's no direct dependency (`.mk` file isn't include), it isn't visible in the build process of the package.

## Solaris ABI

UltraSPARC CPUs are 64-bit processors, but SPARC (Scalable Processor Architecture) has traditionally been designed to support both 32-bit and 64-bit modes. The UltraSPARC I (which is 64-bit) is backward compatible with the earlier 32-bit SPARC V8 architecture, which means that it could run both 32-bit and 64-bit code.

While UltraSPARC processors are 64-bit and support both 32-bit and 64-bit modes, the Solaris ABI is more complex because it needs to manage compatibility between 32-bit and 64-bit binaries. This dual-mode operation and backward compatibility with older software and hardware meant that Solaris had to support different ABIs (for 32-bit and 64-bit applications) and manage their interaction. This is more complicated than on x86_64, where the system was designed from the start with dual-mode operation in mind, making it simpler to run both 32-bit and 64-bit binaries on the same system.

The Solaris ABI is a result of the need to maintain compatibility across many generations of hardware and software, from older 32-bit SPARC systems to the 64-bit UltraSPARC systems that came later.

The best-practices say it's better to build 32-bit applications, but pkgsrc suggests to use ABI=64. But sometimes it's not passing proper options, and Solaris linker fails with ELFCLASS error. It often helps to specify `CFLAGS+=-m64 CXXFLAGS+=-m64`.

## Binutils and versioned symbols

Actually, all packages are built quite well with Solaris binutils, located in `/usr/ccs/bin`.

While building Qt6 (really, Qt6 on Solaris 10 SPARC!) I found out that its plugins don't have a specific Qt-related metadata, and Solaris ld didn't link them correctly, then I needed GNU ld.

How to make GNU ld working:

- LDFLAGS must have `-Wl,-b,elf64-sparc-sol2` (in case of building 64-bit stuff)
- there's a file `libgcc-unwind.map` which is not understood by GNU ld. It must be renamed or moved, but there must be empty file - because gcc relies on it while calling ld.
- it's better to use GNU strip, not Solaris strip (and vice versa: GNU strip doesn't work with Solaris ld)

A part of Solaris ABI is versioned symbols. Solaris uses them to guarantee binary compatibility across system upgrades — critical for enterprise environments where software couldn't always be rebuilt. And I got a problem here.
Solaris ld adds versioned symbols to libraries built by pkgsrc, like this:

```bash
libz.so.1 =>     /usr/pkg/lib/libz.so.1
libz.so.1 (SUNW_1.1) =>  (version not found)
```

This is not an error during runtime, but GNU ld tries to resolve all symbols in all libraries it's linking, why it can cause an error. Especially for `libz.so`, I had to symlink Solaris' `libz.so` to `/usr/pkg/lib` and made GNU ld happy, then I removed the symlink and everything works well.

# too many open files

Some packages (especially glib2 as I remember) try to have a lot of files open and hit Solaris' limit. Solaris has `ulimit` command as usual, but it cannot be more than set in `/etc/system`. And if it's not set, the default is 256.

My settings are:

```bash
set rlim_fd_max=166384
set rlim_fd_cur=32768
```

after changing this file, you need to reboot!


## If something is wrong

But, often packages are successfully built with binutils which the process catches (usually, Solaris `/usr/ccs/bin` ones), but sometimes something is failing. Then:

- use environment variables like AR, AS and try another utility
- if the build process catches a wrong utility, create a symlink / adjust paths
- if there's an error, in most cases Google knows the answer or give you hints
- check and add environment variables (`CFLAGS`, `LDFLAGS`). Sometimes I had to add `-m64` because of Solaris ABI
- libraries cannot be found (or the linker tries to use wrong libraries).
   - check `work/.buildlink` directory for all necessary libraries/header files (and versions) and make symlinks there
   - if it doesn't work, add them to `LDFLAGS` in Makefile, or sometimes it's necessary to patch build scenarios (Makefiles, meson.builds, etc.)
- some functions in Solaris include files are enabled only if `__EXTENSIONS__` is set
- meson can't find correct tools (pkg-config, cmake, etc.). They aren't mentioned in `USE_TOOLS` in Makefile, just add them.
- try to use `libsol10-compat` (see below)
- ...or you have to patch the code/the makefiles/meson build files... In some cases, the compiler suggested me what I needed to use.
- if a package tries to bring another version GCC, just comment USE_FEATURES and GCC_REQD in Makefile and use GCC 9.5 (see below).

And if everything goes well but fails on the linking phase, then

```bash
cd work/sourcename-x.y.z
gmake
```

Just run `gmake` inside the source folder. In some cases, it just run remaining commands and may get done successfully. Or, there will be a visible error which must be fixed. Often these are:

- missing libraries, e.g. `-lsocket -lnsl` (the software assumes them by default or looks for those functions in libc)
- ELFCLASS error (flag `-m64` wasn't passed correctly)

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

**Newer version 0.2.x**: I had to add implementation of `open_memstream` to it, then there's a fork: [https://github.com/merclangrat/libsol10-compat](https://github.com/merclangrat/libsol10-compat). I guess there will be more functions/definitions added (`static_assert` is the first candidate).  
This is not completely my code! ChatGPT helped me to create it.

**TODO** There will be more, because I find more and more things which are not in Solaris 10 and necessary for newer software. Small things, but they cause errors!

### How to use it

It installs `libsol10_compat_patch_pkgsrc` which patches original pkgsrc files and takes two arguments:
- the list of packages to be patched (there's "packages" file in the repo, but you can create your own,
- the root directory of pkgsrc tree, e.g. /usr/pkgsrc (I use /stuff/pkgsrc)

There's a blog post about trying Solaris 10 as a desktop: [https://pekdon.pekwm.se/posts/solaris_desktop/](https://pekdon.pekwm.se/posts/solaris_desktop/) - very useful!

## Getting new certificates

Even if `security/openssl` is installed, it cannot check certificates. There are older ones by OpenCSW in `/etc/opt/csw/ssl/certs`. I guess they can be used but we can install them using pkgsrc, too

I reminded that I had ca-certificates in Linux, but the package from pkgsrc needs `py-cryptography` which needs **Rust** for building, and Rust isn't available for Solaris-SPARC (yet). I couldn't build the older version of it because it uses older OpenSSL (1.0 or 1.1) functions and definitions. But I guess it's possible to patch it...

But it's just necessary to install `security/mozilla-rootcerts-openssl` to get newer certificates. They will be stored in /usr/pkg/etc/openssl/certs, and then everything works well!

## Adding Solaris fonts

When I built `www/netsurf` and started it, there were no letters, only squares. Solaris has its own fonts and we just need to add them to pkgsrc's `fontconfig`, creating `/usr/pkg/etc/fontconfig/local.conf`:

```bash
<?xml version='1.0'?>
<!DOCTYPE fontconfig SYSTEM 'fonts.dtd'>
<fontconfig>
 <dir>/usr/openwin/lib/X11/fonts</dir>
</fontconfig>
```

## The result

The packages I have already built are here: [http://lizaurus.com/solaris10/pkgsrc-solaris10/](http://lizaurus.com/solaris10/pkgsrc-solaris10/)
Almost all of them have been built by gcc5 from OpenCSW and some tools need its `libgcc_s`.

Packages built by gcc9 (see below) are in a subfolder.

After the bootstrap, there're `pkg_*` utilities and those packages can be just installed without rebuilding. My bootstrap is also there: [http://lizaurus.com/solaris10/pkgsrc-solaris10/bootstrap](http://lizaurus.com/solaris10/pkgsrc-solaris10/bootstrap)

To make life simpler, I configure my PATH to look in /usr/pkg first:
`PATH=/usr/pkg/bin:/opt/csw/bin:/usr/sfw/bin:/usr/bin:`

and for root:
`PATH=/usr/pkg/sbin:/opt/csw/sbin:/usr/sfw/sbin:/usr/sbin:/usr/pkg/bin:/opt/csw/bin:/usr/sfw/bin:/usr/bin:`

Unfortunately, I didn't put all my changes into patches (but some of them will be shared soon). I hope my explanations are useful if anyone needs to use pkgsrc on Solaris 10 SPARC in future.

## The system python

I think it's a good way to have `python3` and `python` symlinked to `/usr/pkg/bin/python3.12`. Solaris has Python 3, but much older.

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



## Some issues related to specific packages

### devel/libuv

 patched tcp.c but linking libnbcompat fails, add `-fPIC` flag [https://gnats.netbsd.org/56668](https://gnats.netbsd.org/56668)

### misc/rhash

take newer version (1.4.5) from the next pkgsrc tarball, it builds successfully

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

### databases/shared-mime-info

It has `devel/glib2` as a dependency and uses `libglib-2.0.so`,`libgio-2.0.so`,`libgobject-2.0.so` and `libgmodule-2.0.so`. And the linker doesn't link correctly with `libgmodule-2.0.so`, trying to use the system one, which is very old.  
I had to change `build.ninja` after `bmake configure` - put this library explicitly using the full path.

Also, I had to comment `USE_CXX_FEATURES=filesystem` in Makefile, or it tries to install gcc10 which is not available for Solaris. gcc9 builds it correctly, it has this feature.

### graphics/gdk-pixbuf2

`databases/shared-mime-info` breaks `graphics/gdk-pixbuf2` - it needs `mime.cache` (it took 3 days for me to figure out where the problem was).  
After I fixed it, `gdk-pixbuf2` has a small issue with `systeminfo.h` - just patch the file `pixops.c` to include it unconditionally. And it builds. And it works!

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

### x11/gtk3

Solaris linker (`/usr/ccs/bin/ld`) doesn't support `-Wl,--export-dynamic` - actually, there's a `BUILDLINK_TRANSFORM` option in Makefile to remove it, but `testsuite/gtk/builder` still uses it. Isn't it just a bug? I had to remove it from `ninja.build`.

And `x11/gtk3` has an option `-Dgtk-doc=true` to build a documentation using `xsltproc`. On my Sun Blade 100 with 1.5G of RAM, it was working for 36 hours and couldn't get done.  
I had to stop it and set `-Dgtk-doc` to `false` - but in this case `bmake install` fails. The documentation files are specified in `PLIST`. Of course, there are two options:
- patch `PLIST` and remove everything connected to `gtk-doc`, then package doesn't include the documentation (maybe it's even not needed)
- or, download a binary package for any platform (docs are just html) and put the files into `.work/destdir/usr/pkg/share`.

### www/netsurf

Successfully, the new browser on Solaris! I had to patch some files and add missing libs to `LDFLAGS`.

### textproc/groff

`groff` is a text processor which can be found in each system. pkgsrc's build enables `groff-docs` by default and needs `py-pdf`.  
as it's said on python website, `py-pdf` doesn't need any dependencies in general, but if image extraction is needed, it needs `Pillow` which needs **Rust**. Fail.

I disabled `groff-docs` just running: `bmake install PKG_OPTIONS.groff=-groff-docs CFLAGS+=-D__EXTENSIONS__`  
I guess I will try to build `py-pdf` without Pillow if another package would need it...

### x11/fltk13

pkgsrc has fltk 1.3.9 which I couldn't build using gcc. If I tried Sun Studio, there was an error about ELFCLASS.  
I found a solution. I downloaded fltk 1.3.5 and built it using gcc just by `./configure & gmake` with pkgsrc options, then put to the `work` subdirectory and made a package. Then, the packkage `fltk-1.3.9` actually has version `1.3.5`, but it seems to work...

### net/tigervnc

After `bmake configure`, add `#include <memory.h>` to `./work/tigervnc-1.14.0/unix/vncconfig/QueryConnectDialog.cxx` and `#include <unistd.h>` to `.work/tigervnc-1.14.0/unix/xserver/os/backtrace.c`  
If it fails with linking during X.Org server building, just `cd ./work/tigervnc-1.14.0/unix/xserver/` and run `gmake`. Then, `bmake` will create the package.

But the trickiest part was the font path. TigerVNC builds its own X.Org, and its `./configure` has a parameter to set font paths. But in my case it didn't work.
By default, it sets  
`COMPILEDDEFAULTFONTPATH = ${prefix}/share/fonts/X11/misc/,${prefix}/share/fonts/X11/TTF/,${prefix}/share/fonts/X11/OTF/,${prefix}/share/fonts/X11/Type1/,${prefix}/share/fonts/X11/100dpi/,${prefix}/share/fonts/X11/75dpi/`

I guess it would work, but somehow `${prefix}` wasn't resolved to a variable (it may be because `/bin/sh` is used somewhere which is horribly broken on Solaris).  
I had to replace this line in all Makefiles and `.h` files where I found it, and put
`COMPILEDDEFAULTFONTPATH = /usr/openwin/lib/X11/fonts/misc/,/usr/openwin/lib/X11/fonts/TrueType,/usr/openwin/lib/X11/fonts/Type1/,/usr/openwin/lib/X11/fonts/100dpi/,/usr/openwin/lib/X11/fonts/75dpi/` - to use Solaris fonts.

Also, I needed to install `x11/xsetroot` and `wm/twm` (oh, I haven't tried to run CDE yet in VNC!).

TigerVNC has one more trick. Because SPARC is big-endian, when I try to connect from x86-64 (a little-endian machine), it just shows me a white screen. And on the SPARC side, one of Xvnc threads gets crashed. `vncviewer` on Linux must be run with `-encoding hextile`.

---

Also, I used ChatGPT asking questions and copy-pasting errors. Sometimes it provided too much information, but it could help where to look/patch/fix/etc.


