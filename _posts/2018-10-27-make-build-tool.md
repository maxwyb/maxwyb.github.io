---
layout: post
title:  "Looking into the Make Build Tool & Rethinking Problem-Solving"
date:   2018-10-27 14:35:00 -0700
categories: make, Linux
---
## An unexpected problem
It's been a while and now is time to start blogging again. So last week I tried to help a friend on an assignment of a college software engineering course. The goal is simple: build an old version of `coreutil` from source code, run it to verify that it has a bug, apply a patch file provided, build again and verify the bug is fixed.

Nothing too fancy was going on when we unzipped the distributed tarball and run `./configure --prefix=/home/username/my_coreutil && make && make install`. A customized prefix of installation is added because we don't have root access on the school server to install programs. The patch file basically changes two lines in a source file `ls.c` and added a new file including test cases of this fix:

```diff
---
 NEWS                 |  3 +++
 src/ls.c             |  3 +--
 tests/local.mk       |  1 +
 tests/ls/a-option.sh | 27 +++++++++++++++++++++++++++
 4 files changed, 32 insertions(+), 2 deletions(-)
 create mode 100755 tests/ls/a-option.sh

--- a/NEWS
+++ b/NEWS
  ...some exciting changes...
  
--- a/src/ls.c
+++ b/src/ls.c
@@ -1903,8 +1903,7 @@ decode_switches (int argc, char **argv)
-          if (ignore_mode == IGNORE_DEFAULT)
-            ignore_mode = IGNORE_DOT_AND_DOTDOT;
+          ignore_mode = IGNORE_DOT_AND_DOTDOT;

--- a/tests/local.mk
+++ b/tests/local.mk
@@ -575,6 +575,7 @@ all_tests =		\
   tests/ln/target-1.sh				\
+  tests/ls/a-option.sh				\
   tests/ls/abmon-align.sh			\
   
--- /dev/null
+++ b/tests/ls/a-option.sh
@@ -0,0 +1,27 @@
+#!/bin/sh
  ...some test cases for `ls -a` command...
```

Patching succeeded except for `NEWS` when we run `patch -p1 < patch_file` in the current directory. It may happen if this file was changed later after the patch was generated; let's live with it for now as the documentations would not impact the building process. Surprisingly, it turned out that now we're not able to build anymore:

```bash
$ make
 cd . && /bin/sh /my_dir/coreutils-8.29/build-aux/missing automake-1.15 --gnu Makefile
/u/cs/ugrad/yingbo/cs35l-fall-2018/coreutils-8.29-test/build-aux/missing: line 81: automake-1.15: command not found
WARNING: 'automake-1.15' is missing on your system.
         You should only need it if you modified 'Makefile.am' or
         'configure.ac' or m4 files included by 'configure.ac'.
         The 'automake' program is part of the GNU Automake package:
         <http://www.gnu.org/software/automake>
         It also requires GNU Autoconf, GNU m4 and Perl in order to run:
         <http://www.gnu.org/software/autoconf>
         <http://www.gnu.org/software/m4/>
         <http://www.perl.org/>
make: *** [Makefile.in] Error 127
```

I've been using these commands before when building programs from source on Linux. In most common cases, building is a 3-step process in my understanding: `./configure` detects the current system configurations and generates a `Makefile`, `make` basically follows this bash-script-like file to run commands and build the program, and `make install` copy the compiled executable to the installation paths. Based on this logic, this error then becomes particularly interesting:

1. Why did `make` fail when it had succeeded before applying the patch? *It's the same `Makefile` on the same machine!*  
2. In case if `automake` is really needed during compilation, why didn't `make` fail when we execute it the first time?  

Out of curiosity, we checked that `automake` is even installed and in the right path. There's a difference in minor version (1.13 VS. 1.15) though, but I highly suspect that it matters to build our programs.  
```bash
$ which -a automake
/usr/bin/automake
/usr/local/cs/bin/automake

$ automake --version
automake (GNU automake) 1.13.4
```

Then how about just installing `automake-1.15`? My first thought is that it's probably not expected for us students to install new programs on a school server (we're required to do all assignments here and servers are shared by all engineering students). Of course, we can install it locally and then add the directory to $PATH and/or change our `Makefile` macros to point to the new binary, but I didn't really give it a deep thought considering only the minor version difference and the work involved.  

## Figuring things out
So, a problem with no obvious solution for us, and the exploration began.  

First, the above analysis are based on the premise that *`make`'s behavior should not change when `Makefile` doesn't change*. We were pretty sure this's the case since our patch file doesn't modify `Makefile`, but let's double check by tracking file changes in this directory after every step. `git` seems to be an awesome tool for this; we re-did everything by unzipping the tarball, `git init` to initialize the directory to under `git`, commit everything before applying the patch, and run `git status` & `git diff` to see file differences after applying. No surprise here; nothing changes besides those inside the patch.  

Delving deep into the problem became inevitable at this point, so we started by examining the patch more closely. Besides the new test cases file `a-option.sh` it adds, it also adds one line to `tests/local.mk` to include this new file. This feels like including a new test suite in a top-level "index" of a testing module which is nothing strange, but what exactly is a *.mk* file? A quick Google search shows that it *is* a makefile. Then it's highly suspected that our makefiles are hierarchical; so we checked and `tests/local.mk` is indeed included in our root Makefile above. Thus in this case *our Makefile(s) did change*.  

```makefile
am__DIST_COMMON = $(top_srcdir)/tests/local.mk ...

$(srcdir)/Makefile.in:  $(srcdir)/Makefile.am $(top_srcdir)/tests/local.mk ... $(am__configure_deps)
	@for dep in $?; do \
	  case '$(am__configure_deps)' in \
	    *$$dep*) \
	      $(am__cd) $(srcdir) && $(AUTOMAKE) --gnu \
		&& exit 0; \
	      exit 1;; \
	  esac; \
	  ...
```

Naturally, we moved over to `tests/local.mk` to take a look. Its first line already gave a big hint that this file may be related to the problem:  
```bash
## Process this file with automake to produce Makefile.in -*-Makefile-*-.
```

We had noted these makefile-related files before that look like the following in the project directory, so what are they and how're they related to the `make` build system?  
```bash
[maxwyb@server ~/coreutils-8.29-test]$ ll
total 4240
...
-rw-r--r--  1 yingbo csugrad   54174 Dec 27  2017 aclocal.m4
drwxr-xr-x  2 yingbo csugrad    4096 Oct 27 14:44 build-aux
-rwxr-xr-x  1 yingbo csugrad 1783559 Dec 27  2017 configure
-rw-r--r--  1 yingbo csugrad   23021 Nov 24  2017 configure.ac
rwxr-xr-x  5  yingbo csugrad   57344 Oct 27 14:44 lib
drwxr-xr-x  2 yingbo csugrad   32768 Oct 27 14:44 m4
-rw-r--r--  1 yingbo csugrad    8046 Dec 20  2017 Makefile.am
-rw-r--r--  1 yingbo csugrad  993045 Dec 27  2017 Makefile.in
drwxr-xr-x  2 yingbo csugrad   20480 Oct 27 14:44 man
-rw-r--r--  1 yingbo csugrad  204507 Dec 27  2017 NEWS
drwxr-xr-x  2 yingbo csugrad    8192 Oct 27 14:44 po
-rw-r--r--  1 yingbo csugrad   10771 Dec 23  2017 README
drwxr-xr-x  3 yingbo csugrad   12288 Oct 27 14:44 src
drwxr-xr-x 25 yingbo csugrad    4096 Oct 27 14:44 tests
...
```

A quick search showed that many of these files are used by a set of tools called `autoconf` to generate our actual build scripts. Here, `automake` uses `Makefile.am` to generate `Makefile.in` (*.am* stands for "automake"), and `autoconf` follows `configure.ac` to generate the `configure` script. Then the familiar process discussed above began when `./configure` takes `Makefile.in` and generates `Makefile`. Several layers of abstaction appear to be going on here to generate the build scripts: the first stage includes **scripts that developers write by themselves**, the second stage **longer scripts that are generated automatically on the developers' side**, and the last stage **similarly-long scripts that are generated automatically on the users' side**. We can understand first-second stage translation (`autoconf`) as a sort of boilerplate generation that saves developers' efforts on writing common code, and second-third stage translation (`./configure`) as a necessary step for a user to pre-check his/her local environment before compiling. The following diagram shows this chain of build scripts where blue files are usually generated by the developers and green files by users.  

![make-build-system-diagram](/assets/make-build-system-1.png "make-build-system-diagram")

So everything began to make sense now: we triggered a file change in the first stage of building by modifying `tests/local.mk`, so all second and third stage scripts have to be regenerated. The first-second stage translation definitely needs `automake` version 1.15. (by playing around with `build-aux/missing` binary detection script with other installed executables, we found version is part of the binary check as well.) As noted in [GNU's automake manual](https://www.gnu.org/software/automake/manual/html_node/Creating-amhello.html#Creating-amhello):  
> Note that running autoreconf is only needed initially when the GNU Build System does not exist. When you later change some instructions in a Makefile.am or configure.ac, the relevant part of the build system will be regenerated automatically when you execute make.

Going back into the code block of `Makefile` included above, we see that `tests/local.am` is a dependency inside a target, which does some work for each dependency involving `$(AUTOMAKE)` when one or more dependency updates. Therefore, **there're nuances of how `make` works that we overlooked before**: `make` does follow & only follow and `Makefile` provided to it, but this dependency check allows it to detect any file changes in lower stage, start re-building auto-generated files (perhaps including itself) and thus can modify scripts among the entire build chain (referred by the yellow arrow in the diagram). [This question](https://stackoverflow.com/questions/1789705/how-does-make-know-which-files-to-update) explains how `make` keeps track of what files in the project directory have changed since last build (and shows why an accurate system clock is so important). Note that since second-stage build scripts (`configure`, `Makefile.in`) are included in our distributed tarball, there's no need to run first-second layer generation before files are changed. This explains why `automake` is not called when we run `make` before applying the patch so it succeeded.  

## Solution
The solution became simple after discovering the root cause: install `automake-1.15` to a userland path where we have permission, add it to $PATH environment variable and re-run `make`.  

```bash
$ tar xf automake-1.15.tar.xz
$ ./configure --prefix=/my/userland/directory
$ make
$ make install
$ export PATH=/my/userland/directory:$PATH
$ which -a automake  # make sure our new binary is there
```

Later, we dived a little deeper into the `autoconf` tools by walking over [an example in GNU manual](https://www.gnu.org/software/automake/manual/html_node/Creating-amhello.html#Creating-amhello) to set up our own build system for a Hello World project. [This GitHub repo](https://github.com/maxwyb/autoconf-Demo) contains the source code. Each branch represents a stage translation so you can simply examine file modifications by doing `git diff autoreconf configure`, for example. The boilerplate generator indeed saves developers' times - developers only need to write 13 lines of code in first-stage scripts (`Makefile.am` and `configure.ac`). The tools generate build scripts later that're all 800+ lines of code each.  

## Reflections
The problem we encountered itself shouldn't be too technically challenging; any developer frequently using the `make` build system should be able to figure it out in seconds. It's the thoughts and approaches to the problem that I find intersting to be documented. In the software engineering world, there can be two aspects in problem-solving in general:  

1. Understand what had happened & why it happened
2. Know what to do to fix the problem

The first usually leads to the second naturally but the second doesn't necessarily mean the first. (and here comes the comic) ![make-build-system-comic](/assets/make-build-system-2.png "make-build-system-comic") In our problem, we may choose to install the correct version of `automake` in the first place and problem would be solved, but then may never know what really happened beneath the surface. There're so many cool things to learn anyways, aren't there?  

## References
[Building a GNU Autotools Project](http://inti.sourceforge.net/tutorial/libinti/autotoolsproject.html)  
[What are Makefile.am and Makefile.in?](https://stackoverflow.com/questions/2531827/what-are-makefile-am-and-makefile-in)
