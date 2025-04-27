# Some issues related to specific packages

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


## devel/libuv

 patched tcp.c but linking libnbcompat fails, add `-fPIC` flag [https://gnats.netbsd.org/56668](https://gnats.netbsd.org/56668)

## misc/rhash

take newer version (1.4.5) from the next pkgsrc tarball, it builds successfully

## devel/glib2

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

## devel/cmake

cmake needs -lncurses because it tries to link itself with Solaris curses library + new gcc

I removed two files connected to GHS (they caused error about `gbuild` from Green Hills software, I guess we don't need that), also `bmake package SSP_SUPPORTED=no` (I think it can be solved by another way, for example symlinking `/usr/local/lib/sparcv9/libssp.so` to `/usr/pkg/lib`).

How long did it take to build cmake? Here it is:

```bash
real    786m29.379s
user    708m1.986s
sys     44m31.926s
```

it's built with gcc 9.5 and needs libs from the archive (see [GCC 9](#gcc-9) (gcc5 could't build it).

## databases/shared-mime-info

It has `devel/glib2` as a dependency and uses `libglib-2.0.so`,`libgio-2.0.so`,`libgobject-2.0.so` and `libgmodule-2.0.so`. And the linker doesn't link correctly with `libgmodule-2.0.so`, trying to use the system one, which is very old.  
I had to change `build.ninja` after `bmake configure` - put this library explicitly using the full path.

Also, I had to comment `USE_CXX_FEATURES=filesystem` in Makefile, or it tries to install gcc10 which is not available for Solaris. gcc9 builds it correctly, it has this feature.

## graphics/gdk-pixbuf2

`databases/shared-mime-info` breaks `graphics/gdk-pixbuf2` - it needs `mime.cache` (it took 3 days for me to figure out where the problem was).  
After I fixed it, `gdk-pixbuf2` has a small issue with `systeminfo.h` - just patch the file `pixops.c` to include it unconditionally. And it builds. And it works!

## devel/pango

Building pango was tricky. There were linker problems, and because I was not successful to use `gld`, I had to replace `-z defs` to `-z nodefs` in meson build (also, after initial configure).
`pango` and `cairo` seem to work well because I could build `rxvt-unicode` and there's a nice terminal! Woohoo!

## graphics/netpbm

`netpbm` mysteriously has just `cc` without paths in `config.mk.in`, and it doesn't get changed by configure scripts. Putting `cc` symlink somewhere to `/usr/pkg/bin` didn't help, I had to change the path in the file and run bmake again.

## security/gnutls

Oh, that was the trickiest one! I don't know why building via `bmake` was failing, because if I used `gmake` in the source directory, it worked. I had to build it like this, but then use `readelf` and `elfedit` to fix binaries because there were relative paths.  
I think to try again and figure out what was happening there to make it buildable via `bmake` properly.

## graphics/MesaLib

It uses `static_assert` which is not available in Solaris, I patched the code and could build it.

## x11/gtk3

Solaris linker (`/usr/ccs/bin/ld`) doesn't support `-Wl,--export-dynamic` - actually, there's a `BUILDLINK_TRANSFORM` option in Makefile to remove it, but `testsuite/gtk/builder` still uses it. Isn't it just a bug? I had to remove it from `ninja.build`.

And `x11/gtk3` has an option `-Dgtk-doc=true` to build a documentation using `xsltproc`. On my Sun Blade 100 with 1.5G of RAM, it was working for 36 hours and couldn't get done.  
I had to stop it and set `-Dgtk-doc` to `false` - but in this case `bmake install` fails. The documentation files are specified in `PLIST`. Of course, there are two options:
- patch `PLIST` and remove everything connected to `gtk-doc`, then package doesn't include the documentation (maybe it's even not needed)
- or, download a binary package for any platform (docs are just html) and put the files into `.work/destdir/usr/pkg/share`.

## www/netsurf

Successfully, the new browser on Solaris! I had to patch some files and add missing libs to `LDFLAGS`.

## textproc/groff

`groff` is a text processor which can be found in each system. pkgsrc's build enables `groff-docs` by default and needs `py-pdf`.  
as it's said on python website, `py-pdf` doesn't need any dependencies in general, but if image extraction is needed, it needs `Pillow` which needs **Rust**. Fail.

I disabled `groff-docs` just running: `bmake install PKG_OPTIONS.groff=-groff-docs CFLAGS+=-D__EXTENSIONS__`  
I guess I will try to build `py-pdf` without Pillow if another package would need it...

## x11/fltk13

pkgsrc has fltk 1.3.9 which I couldn't build using gcc. If I tried Sun Studio, there was an error about ELFCLASS.  
I found a solution. I downloaded fltk 1.3.5 and built it using gcc just by `./configure & gmake` with pkgsrc options, then put to the `work` subdirectory and made a package. Then, the packkage `fltk-1.3.9` actually has version `1.3.5`, but it seems to work...

## net/tigervnc

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

