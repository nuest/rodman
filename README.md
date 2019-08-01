# rodman

Examples, tests, and documentation for using Rocker with [podman](https://podman.io/).

## Previous work and background

- Discussion in Fedora forum about running RStudio with podman: https://discussion.fedoraproject.org/t/integrate-r-with-openblas-in-fedora/1052/22
  - related GitHub issue: https://github.com/rocker-org/rocker/issues/202
- https://www.heise.de/developer/artikel/Podman-Linux-Container-einfach-gemacht-Teil-3-4476343.html
- https://mkdev.me/en/posts/dockerless-part-3-moving-development-environment-to-containers-with-podman

## Example of `podman` advantage

Consider the following demonstration of the advantage of running containers without root.

On the test machine, create a text file belonging to user `root`:

```bash
daniel@gin-nuest:~/$ sudo touch /tmp/root.txt
[sudo] password for daniel: 
daniel@gin-nuest:~/$ ll /tmp/root.txt 
-rw-r--r-- 1 root root 0 Aug  1 12:13 /tmp/root.txt
daniel@gin-nuest:~/$ touch /tmp/root.txt 
touch: cannot touch '/tmp/root.txt': Permission denied
```

The regular user cannot touch the file (but they can read it).

Start a Docker container, mount the `/tmp/` directory, and append a text string to the file:

```bash
daniel@gin-nuest:~/git/rocker-versioned/rstudio/3.6.0$ docker run --rm -it -v /tmp:/tempdir rocker/r-ver
[...]
> file.info("/tempdir/root.txt")
                  size isdir mode               mtime               ctime
/tempdir/root.txt   21 FALSE  644 2019-08-01 10:37:56 2019-08-01 10:37:56
                                atime uid gid uname grname
/tempdir/root.txt 2019-08-01 10:37:58   0   0  root   root
> file.show(file = "/tempdir/root.txt")

> cat("From Docker container", file = "/tempdir/root.txt")
> file.show(file = "/tempdir/root.txt")
From Docker container
> 
```

_This works!_ The container can write the image, because the user running the container is the Docker daemon.

Start a container with `podman` as a regular user, mount the `/tmp` directory, and attempt to append to the file:

```bash
daniel@gin-nuest:~/git/rocker-versioned/rstudio/3.6.0$ podman run --rm -it -v /tmp:/tempdir docker.io/rocker/r-ver
[...]
> file.info("/tempdir/root.txt")
                  size isdir mode               mtime               ctime
/tempdir/root.txt   21 FALSE  644 2019-08-01 10:37:56 2019-08-01 10:37:56
                                atime   uid   gid  uname  grname
/tempdir/root.txt 2019-08-01 10:37:58 65534 65534 nobody nogroup
> cat("From podman container", file = "/tempdir/root.txt")
Error in file(file, ifelse(append, "a", "w")) : 
  cannot open the connection
In addition: Warning message:
In file(file, ifelse(append, "a", "w")) :
  cannot open file '/tempdir/root.txt': Permission denied
> system2("whoami")
root
> 
```

_This does not work!_ Besides the user in the container being `root`, the in-container `root` is mapped to a the regular user outside of the container.

## Building Rocker images with podman

To build images with `podman`, the image names must pre prepended with the default registry, which is not automatically activated on other tools than `docker`, and potentially the default `library` organisation.

`debian:stretch` becomes `docker.io/library/debian:stretch`.

Three exemplary Dockerfiles and auxiliary files have been added to this repository from https://github.com/rocker-org/rocker-versioned, keeping the directory structure, see `./r-ver/..`.

### Building `r-ver` with `podman`

```bash
# using second run of command and cached layers for brevity of output:
daniel@gin-nuest:~/git/rodman/r-ver/3.6.0$ podman build --tag rodman/r-ver:3.6.0 .
STEP 1: FROM docker.io/library/debian:stretch
STEP 2: LABEL org.label-schema.license="GPL-2.0"       org.label-schema.vcs-url="https://github.com/rocker-org/rocker-versioned"       org.label-schema.vendor="Rocker Project"       maintainer="Carl Boettiger <cboettig@ropensci.org>"
--> Using cache d85247d22b45de994b7740063e2716a8922bb2110f6c7466638bc63495eeea7d
STEP 3: ARG R_VERSION
--> Using cache 430c7d000310336503593ed764e53bef1458bec6eeabf69ed32aa294fc11f741
STEP 4: ARG BUILD_DATE
--> Using cache 4eecba5017581b9740d0659d880ef845c8e8065088e1068ea2f08f7378e9fb24
STEP 5: ENV BUILD_DATE ${BUILD_DATE:-2019-07-05}
--> Using cache 193cbcb395a9528daca52b60bac5b515a97d851aaca0bf7f97cd0cc2a696aeb3
STEP 6: ENV R_VERSION=${R_VERSION:-3.6.0}     LC_ALL=en_US.UTF-8     LANG=en_US.UTF-8     TERM=xterm
--> Using cache 62e41f19f55f767aff4599471c343ee43a7445d284ec21d631b58baa3de24ee9
STEP 7: RUN apt-get update   && apt-get install -y --no-install-recommends     bash-completion     ca-certificates     file     fonts-texgyre     g++     gfortran     gsfonts     libblas-dev     libbz2-1.0     libcurl3     libicu57     libjpeg62-turbo     libopenblas-dev     libpangocairo-1.0-0     libpcre3     libpng16-16     libreadline7     libtiff5     liblzma5     locales     make     unzip     zip     zlib1g   && echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen   && locale-gen en_US.utf8   && /usr/sbin/update-locale LANG=en_US.UTF-8   && BUILDDEPS="curl     default-jdk     libbz2-dev     libcairo2-dev     libcurl4-openssl-dev     libpango1.0-dev     libjpeg-dev     libicu-dev     libpcre3-dev     libpng-dev     libreadline-dev     libtiff5-dev     liblzma-dev     libx11-dev     libxt-dev     perl     tcl8.6-dev     tk8.6-dev     texinfo     texlive-extra-utils     texlive-fonts-recommended     texlive-fonts-extra     texlive-latex-recommended     x11proto-core-dev     xauth     xfonts-base     xvfb     zlib1g-dev"   && apt-get install -y --no-install-recommends $BUILDDEPS   && cd tmp/   && curl -O https://cran.r-project.org/src/base/R-3/R-${R_VERSION}.tar.gz   && tar -xf R-${R_VERSION}.tar.gz   && cd R-${R_VERSION}   && R_PAPERSIZE=letter     R_BATCHSAVE="--no-save --no-restore"     R_BROWSER=xdg-open     PAGER=/usr/bin/pager     PERL=/usr/bin/perl     R_UNZIPCMD=/usr/bin/unzip     R_ZIPCMD=/usr/bin/zip     R_PRINTCMD=/usr/bin/lpr     LIBnn=lib     AWK=/usr/bin/awk     CFLAGS="-g -O2 -fstack-protector-strong -Wformat -Werror=format-security -Wdate-time -D_FORTIFY_SOURCE=2 -g"     CXXFLAGS="-g -O2 -fstack-protector-strong -Wformat -Werror=format-security -Wdate-time -D_FORTIFY_SOURCE=2 -g"   ./configure --enable-R-shlib                --enable-memory-profiling                --with-readline                --with-blas                --with-tcltk                --disable-nls                --with-recommended-packages   && make   && make install   && echo "options(repos = c(CRAN = 'https://cran.rstudio.com/'), download.file.method = 'libcurl')" >> /usr/local/lib/R/etc/Rprofile.site   && mkdir -p /usr/local/lib/R/site-library   && chown root:staff /usr/local/lib/R/site-library   && chmod g+wx /usr/local/lib/R/site-library   && echo "R_LIBS_USER='/usr/local/lib/R/site-library'" >> /usr/local/lib/R/etc/Renviron   && echo "R_LIBS=\${R_LIBS-'/usr/local/lib/R/site-library:/usr/local/lib/R/library:/usr/lib/R/library'}" >> /usr/local/lib/R/etc/Renviron   && [ -z "$BUILD_DATE" ] && BUILD_DATE=$(TZ="America/Los_Angeles" date -I) || true   && MRAN=https://mran.microsoft.com/snapshot/${BUILD_DATE}   && echo MRAN=$MRAN >> /etc/environment   && export MRAN=$MRAN   && echo "options(repos = c(CRAN='$MRAN'), download.file.method = 'libcurl')" >> /usr/local/lib/R/etc/Rprofile.site   && Rscript -e "install.packages(c('littler', 'docopt'), repo = '$MRAN')"   && ln -s /usr/local/lib/R/site-library/littler/examples/install2.r /usr/local/bin/install2.r   && ln -s /usr/local/lib/R/site-library/littler/examples/installGithub.r /usr/local/bin/installGithub.r   && ln -s /usr/local/lib/R/site-library/littler/bin/r /usr/local/bin/r   && cd /   && rm -rf /tmp/*   && apt-get remove --purge -y $BUILDDEPS   && apt-get autoremove -y   && apt-get autoclean -y   && rm -rf /var/lib/apt/lists/*
--> Using cache 73f2ef360f86bc1f8b9ce0f75396d32e3d9827913b401b02e087c22508fc9ed5
STEP 8: CMD ["R"]
--> Using cache 773c54fe0fe26978d177a31980777bdbfbe7b24319cbc188d38f45330211b771
STEP 9: COMMIT rodman/r-ver:3.6.0
773c54fe0fe26978d177a31980777bdbfbe7b24319cbc188d38f45330211b771
```

**Works** after fixing the base image by adding the full registry.

### Building `rocker/rstudio` with podman

```
aniel@gin-nuest:~/git/rodman$ podman build --tag rodman/rstudio:3.6.0 --file rstudio/3.6.0/Dockerfile.rodman rstudio/3.6.0/
STEP 1: FROM docker.io/rocker/r-ver:3.6.0
STEP 2: ARG RSTUDIO_VERSION
--> Using cache cfcdd3bb66e551b33f073b79cbff202bf460b4c0b1e99e46ed99f9dadce410cc
STEP 3: ENV RSTUDIO_VERSION=${RSTUDIO_VERSION:-1.2.1335}
--> Using cache d90b44ff2463a33bf1369f2238c101bbfeaa5e2288f605200ea5298de3b35064
STEP 4: ARG S6_VERSION
--> Using cache d6552a44e41edc9fb70febb38a4b5ea6f539ae5a678f789369764cb03d767432
STEP 5: ARG PANDOC_TEMPLATES_VERSION
--> Using cache 8c1d9b7ccff5a6a7744af1df271310da7f11a85caa4d9e0dd1ef6d853a663a61
STEP 6: ENV S6_VERSION=${S6_VERSION:-v1.21.7.0}
--> Using cache bdf3741124e5bf99fe1cdade9a4951095a35d7557f205a422dcc5a5a01fbe453
STEP 7: ENV S6_BEHAVIOUR_IF_STAGE2_FAILS=2
--> Using cache 6442040653985f319f71199450354f075219aaa626bc95b15a80c7efcfb6a102
STEP 8: ENV PATH=/usr/lib/rstudio-server/bin:$PATH
--> Using cache a2b67333856079202d94d0bdde2b6fba33e420996aa0a2ed204f351195f29baf
STEP 9: ENV PANDOC_TEMPLATES_VERSION=${PANDOC_TEMPLATES_VERSION:-2.6}
--> Using cache 1cf0bb4e4da5fbf3014fa538957fc873fc8f8e0f0b06562ad1419ff11857e4ab
STEP 10: RUN apt-get update   && apt-get install -y --no-install-recommends     file     git     libapparmor1     libcurl4-openssl-dev     libedit2     libssl-dev     lsb-release     psmisc     procps     python-setuptools     sudo     wget     libclang-dev     libclang-3.8-dev     libobjc-6-dev     libclang1-3.8     libclang-common-3.8-dev     libllvm3.8     libobjc4     libgc1c2   && if [ -z "$RSTUDIO_VERSION" ]; then RSTUDIO_URL="https://www.rstudio.org/download/latest/stable/server/debian9_64/rstudio-server-latest-amd64.deb"; else RSTUDIO_URL="http://download2.rstudio.org/server/debian9/x86_64/rstudio-server-${RSTUDIO_VERSION}-amd64.deb"; fi   && wget -q $RSTUDIO_URL   && dpkg -i rstudio-server-*-amd64.deb   && rm rstudio-server-*-amd64.deb   && ln -s /usr/lib/rstudio-server/bin/pandoc/pandoc /usr/local/bin   && ln -s /usr/lib/rstudio-server/bin/pandoc/pandoc-citeproc /usr/local/bin   && git clone --recursive --branch ${PANDOC_TEMPLATES_VERSION} https://github.com/jgm/pandoc-templates   && mkdir -p /opt/pandoc/templates   && cp -r pandoc-templates*/* /opt/pandoc/templates && rm -rf pandoc-templates*   && mkdir /root/.pandoc && ln -s /opt/pandoc/templates /root/.pandoc/templates   && apt-get clean   && rm -rf /var/lib/apt/lists/   && mkdir -p /etc/R   && echo '\n    \n# Configure httr to perform out-of-band authentication if HTTR_LOCALHOST     \n# is not set since a redirect to localhost may not work depending upon     \n# where this Docker container is running.     \nif(is.na(Sys.getenv("HTTR_LOCALHOST", unset=NA))) {     \n  options(httr_oob_default = TRUE)     \n}' >> /usr/local/lib/R/etc/Rprofile.site   && echo "PATH=${PATH}" >> /usr/local/lib/R/etc/Renviron   && useradd rstudio   && echo "rstudio:rstudio" | chpasswd   && mkdir /home/rstudio  && chown rstudio:rstudio /home/rstudio  && addgroup rstudio staff   &&  echo 'rsession-which-r=/usr/local/bin/R' >> /etc/rstudio/rserver.conf   && echo 'lock-type=advisory' >> /etc/rstudio/file-locks   && git config --system credential.helper 'cache --timeout=3600'   && git config --system push.default simple   && wget -P /tmp/ https://github.com/just-containers/s6-overlay/releases/download/${S6_VERSION}/s6-overlay-amd64.tar.gz   && tar xzf /tmp/s6-overlay-amd64.tar.gz -C /   && mkdir -p /etc/services.d/rstudio   && echo '#!/usr/bin/with-contenv bash           \n## load /etc/environment vars first:                 \n for line in $( cat /etc/environment ) ; do export $line ; done           \n exec /usr/lib/rstudio-server/bin/rserver --server-daemonize 0'           > /etc/services.d/rstudio/run   && echo '#!/bin/bash           \n rstudio-server stop'           > /etc/services.d/rstudio/finish   && mkdir -p /home/rstudio/.rstudio/monitored/user-settings   && echo 'alwaysSaveHistory="0"           \nloadRData="0"           \nsaveAction="0"'           > /home/rstudio/.rstudio/monitored/user-settings/user-settings   && chown -R rstudio:rstudio /home/rstudio/.rstudio
--> Using cache 46ac57b02b3ca15d6fdee74f6e77b142a986fdf6d23958006ec1928d8f8861b9
STEP 11: COPY userconf.sh /etc/cont-init.d/userconf
--> Using cache e970cc193ee88024be654662b8a989a3160845d47d49aba5600384f254969d52
STEP 12: COPY add_shiny.sh /etc/cont-init.d/add
--> Using cache 7c5de0eb2728e40d1f2aa52805379012a9dcf20d6259a61b40aa7c07f7121489
STEP 13: COPY disable_auth_rserver.conf /etc/rstudio/disable_auth_rserver.conf
--> Using cache cbf5b55bc224c36612fbe875af30338a043bcd784a523ce245176f1f8efde398
STEP 14: COPY pam-helper.sh /usr/lib/rstudio-server/bin/pam-helper
--> Using cache 0eede4eb3e7cd6b2defa5650fab435513f19612a47e646b3ff4e7fb3e9ebec48
STEP 15: EXPOSE 8787
--> Using cache f2ba6ce67ad076092cedd167d8a78d7eb0ebb252b04ee52eb8fc0b4bba7e4189
STEP 16: VOLUME /home/rstudio/kitematic
--> Using cache 552ab48d365d0f6d56a7b7c981ffffd84f79aa2f648aa8962d080e9fb5a73933
STEP 17: CMD ["/init"]
--> Using cache 305a768e37150581a0afa4a861bc9dc2cbe992f912367f24d7b5fe8138ce404c
STEP 18: COMMIT rodman/rstudio:3.6.0
305a768e37150581a0afa4a861bc9dc2cbe992f912367f24d7b5fe8138ce404c
```

## Running Rocker images with podman

**Works** for `r-base` and `r-ver` where only R prompt is started:

```
daniel@gin-nuest:~/git/rocker-versioned/rstudio/3.6.0$ podman pull docker.io/rocker/r-base
Trying to pull docker.io/rocker/r-base...Getting image source signatures
Copying blob 52d0acf1b567 done
Copying blob 23427ac613ac done
Copying blob 0ec668fb1856 done
Copying blob 877fbe938c96 done
Copying blob badc1fe0e63e done
Copying blob a53b6144c81d done
Copying config cfa376aee5 done
Writing manifest to image destination
Storing signatures
cfa376aee59352c54ab5028e9d69832daa6be6c30ccb67a54efc5375e7bd9efa
daniel@gin-nuest:~/git/rocker-versioned/rstudio/3.6.0$ podman run --rm -it docker.io/rocker/r-base

R version 3.6.1 (2019-07-05) -- "Action of the Toes"
Copyright (C) 2019 The R Foundation for Statistical Computing
Platform: x86_64-pc-linux-gnu (64-bit)

R is free software and comes with ABSOLUTELY NO WARRANTY.
You are welcome to redistribute it under certain conditions.
Type 'license()' or 'licence()' for distribution details.

  Natural language support but running in an English locale

R is a collaborative project with many contributors.
Type 'contributors()' for more information and
'citation()' on how to cite R or R packages in publications.

Type 'demo()' for some demos, 'help()' for on-line help, or
'help.start()' for an HTML browser interface to help.
Type 'q()' to quit R.

> sessionInfo()
R version 3.6.1 (2019-07-05)
Platform: x86_64-pc-linux-gnu (64-bit)
Running under: Debian GNU/Linux 10 (buster)

Matrix products: default
BLAS:   /usr/lib/x86_64-linux-gnu/blas/libblas.so.3.8.0
LAPACK: /usr/lib/x86_64-linux-gnu/lapack/liblapack.so.3.8.0

locale:
 [1] LC_CTYPE=en_US.UTF-8       LC_NUMERIC=C              
 [3] LC_TIME=en_US.UTF-8        LC_COLLATE=en_US.UTF-8    
 [5] LC_MONETARY=en_US.UTF-8    LC_MESSAGES=en_US.UTF-8   
 [7] LC_PAPER=en_US.UTF-8       LC_NAME=C                 
 [9] LC_ADDRESS=C               LC_TELEPHONE=C            
[11] LC_MEASUREMENT=en_US.UTF-8 LC_IDENTIFICATION=C       

attached base packages:
[1] stats     graphics  grDevices utils     datasets  methods   base     

loaded via a namespace (and not attached):
[1] compiler_3.6.1
> 
```

_Does not work for images starting RStudio:_

```bash
daniel@gin-nuest:~$ podman run -p 8787:8787 -e PASSWORD=rockman --rm -it docker.io/rocker/verse
[s6-init] making user provided files available at /var/run/s6/etc...exited 0.
[s6-init] ensuring user provided files have correct perms...exited 0.
[fix-attrs.d] applying ownership & permissions fixes...
[fix-attrs.d] done.
[cont-init.d] executing container initialization scripts...
[cont-init.d] add: executing... 
Nothing additional to add
[cont-init.d] add: exited 0.
[cont-init.d] userconf: executing... 
[cont-init.d] userconf: exited 0.
[cont-init.d] done.
[services.d] starting services
s6-supervise (child): fatal: unable to exec run: Permission denied
s6-supervise rstudio: warning: unable to spawn ./run - waiting 10 seconds
[services.d] done.
s6-supervise (child): fatal: unable to exec run: Permission denied
s6-supervise rstudio: warning: unable to spawn ./run - waiting 10 seconds
s6-supervise (child): fatal: unable to exec run: Permission denied
s6-supervise rstudio: warning: unable to spawn ./run - waiting 10 seconds
s6-supervise (child): fatal: unable to exec run: Permission denied
s6-supervise rstudio: warning: unable to spawn ./run - waiting 10 seconds
s6-supervise (child): fatal: unable to exec run: Permission denied
```

`R` works:

```
daniel@gin-nuest:~/git/rodman$ podman run --rm -it rodman/rstudio:3.6.0 R

R version 3.6.0 (2019-04-26) -- "Planting of a Tree"
Copyright (C) 2019 The R Foundation for Statistical Computing
Platform: x86_64-pc-linux-gnu (64-bit)

R is free software and comes with ABSOLUTELY NO WARRANTY.
You are welcome to redistribute it under certain conditions.
Type 'license()' or 'licence()' for distribution details.

R is a collaborative project with many contributors.
Type 'contributors()' for more information and
'citation()' on how to cite R or R packages in publications.

Type 'demo()' for some demos, 'help()' for on-line help, or
'help.start()' for an HTML browser interface to help.
Type 'q()' to quit R.

> library("rstudioapi")
Error in library("rstudioapi") : there is no package called ‘rstudioapi’
> installed.packages()
           Package      LibPath                         Version   
docopt     "docopt"     "/usr/local/lib/R/site-library" "0.6.1"   
littler    "littler"    "/usr/local/lib/R/site-library" "0.3.8"   
base       "base"       "/usr/local/lib/R/library"      "3.6.0"   
boot       "boot"       "/usr/local/lib/R/library"      "1.3-22"  
class      "class"      "/usr/local/lib/R/library"      "7.3-15"  
cluster    "cluster"    "/usr/local/lib/R/library"      "2.0.8"   
codetools  "codetools"  "/usr/local/lib/R/library"      "0.2-16"  
compiler   "compiler"   "/usr/local/lib/R/library"      "3.6.0"
[...]
```

## Potential next steps

- Use explicit library and registry in Dockerfiles: 
- Add podman building of images to Travis CI tests to make sure there are no problems building the images with non-Docker tools
- Fix `rocker/rstudio`, see https://github.com/rocker-org/rocker/issues/202

## License

Copyright Daniel Nüst, published under GPL v. 3.0
