This guide provides instructions and support files to run the AFLnet fuzzer on the forked-daapd network server, on Ubuntu 18.04.2 LTS.

The guide assumes that AFLnet has already been compiled according to its [tutorial](https://github.com/aflnet/aflnet#tutorial---fuzzing-live555-media-streaming-server), and that environmental variables (e.g., `$AFLNET`) are set accordingly.

Moreover, the guide assumes that the `$WORKDIR` variable has been set to the folder with a copy of this repository, as follows:

```
git clone https://github.com/rnatella/aflnet-forked-daapd.git
cd aflnet-forked-daapd
export WORKDIR=$(pwd)
```


AFLnet must be re-compiled with support to parse HTTP requests and responses:

```
cd $AFLNET
patch -p1 < $WORKDIR/aflnet-forked-daapd.patch
make
```


Before fuzzing, we compile and test the plain forked-daapd server (these instructions are from [INSTALL.md in forked-daapd](https://github.com/ejurgensen/forked-daapd/blob/master/INSTALL.md)):


```
sudo apt-get install \
  build-essential git autotools-dev autoconf automake libtool gettext gawk \
  gperf antlr3 libantlr3c-dev libconfuse-dev libunistring-dev libsqlite3-dev \
  libavcodec-dev libavformat-dev libavfilter-dev libswscale-dev libavutil-dev \
  libasound2-dev libmxml-dev libgcrypt20-dev libavahi-client-dev zlib1g-dev \
  libevent-dev libplist-dev libsodium-dev libjson-c-dev libwebsockets-dev \
  libcurl*-dev


cd $WORKDIR

git clone https://github.com/ejurgensen/forked-daapd.git

cd forked-daapd

autoreconf -i
./configure --prefix=/usr --sysconfdir=/etc --localstatedir=/var
make
sudo make install
```


To configure the server, edit the configuration file to configure the folder with an MP3 library, such as:

```
perl -p -i -e 's|/srv/music|'$WORKDIR'/MP3|' forked-daapd.conf
```


Finally, to launch the server:

```
/usr/sbin/forked-daapd -d 5 -c ./forked-daapd.conf -f
```

You can test the server with a browser at http://localhost:3689 .



Before fuzzing, we recompile the server (and some of its dependencies) to avoid multi-threading, in order to improve stability. We replace the pthreads library with the GNU Pth library (as suggested in the AFL user guide), which provides user-space threading and wrappers for the pthreads library.

Since the GNU Pth package in Ubuntu was not compiled with the pthread wrappers, we will compile the library from source code.

```
cd $WORKDIR

wget ftp://ftp.gnu.org/gnu/pth/pth-2.0.7.tar.gz

tar zxf pth-2.0.7.tar.gz

cd pth-2.0.7

./configure --enable-pthread

make
```


We then need to recompile two dependencies (libevent and libwebsockets) in order to use I/O system calls that are compatible with GNU Pth. We patch the library to use APIs from GNU Pth.

To recompile libevent:


```
cd $WORKDIR

git clone https://github.com/libevent/libevent.git

cd libevent

patch -p1 < $WORKDIR/libevent.patch

mkdir build
cd build

cmake -DEVENT__DISABLE_TESTS=ON -DEVENT__DISABLE_SAMPLES=ON -DEVENT__DISABLE_REGRESS=ON -DEVENT__DISABLE_BENCHMARK=ON ..
```


Edit the file `include/event2/event-config.h` to comment (undefine) all features containing the keywords `POLL`, `EPOLL`, `DEVPOLL`, `KQUEUE`, `EVENTFD`.

```
perl -p -i -e 'if(/POLL|EVENTFD|KQUEUE/) { $_ = "//".$_; }' include/event2/event-config.h
```

Then, check that the features for `EVENT__HAVE_SELECT` and `EVENT__HAVE_SYS_SELECT_H` are defined. These features will appear as follows:

```
/* Define to 1 if you have the `select' function. */
#define EVENT__HAVE_SELECT 1

/* Define to 1 if you have the <sys/select.h> header file. */
#define EVENT__HAVE_SYS_SELECT_H 1
```

To compile the project:
```
make
```


To recompile libwebsockets:


```
cd $WORKDIR

wget https://launchpad.net/ubuntu/+archive/primary/+sourcefiles/libwebsockets/2.0.3-3/libwebsockets_2.0.3.orig.tar.gz

tar zxf libwebsockets_2.0.3.orig.tar.gz

cd libwebsockets-2.0.3

patch -p1 < $WORKDIR/libwebsockets.patch

mkdir build
cd build

cmake -DLWS_WITHOUT_TESTAPPS=ON -DLWS_WITHOUT_TEST_CLIENT=ON -DLWS_WITHOUT_TEST_ECHO=ON -DLWS_WITHOUT_TEST_FRAGGLE=ON -DLWS_WITHOUT_TEST_PING=ON -DLWS_WITHOUT_TEST_SERVER=ON -DLWS_WITHOUT_TEST_SERVER_EXTPOLL=ON ..

make
```


Finally, we recompile the forked-daapd. We patch the source code to set the seed for random numbers to a fixed value, and to appen the characters "\r\n" to HTTP replies from forked-daapd, to make it easier for AFLnet to extract responses. We also disable unnecessary dependencies.

```
cd $WORKDIR

cd forked-daapd

patch -p1 < ../forked-daapd.patch

CC=$AFLNET/afl-clang-fast LD=$AFLNET/aflnet/afl-clang-fast CFLAGS="-DSQLITE_CORE" LDFLAGS="-L$WORKDIR/libevent/build/lib/ -L$WORKDIR/pth-2.0.7/.libs/ -L$WORKDIR/libwebsockets-2.0.3/build/lib/ -lpth"    ./configure --prefix=/usr --sysconfdir=/etc --localstatedir=/var --disable-shared  --disable-mpd --disable-itunes --disable-lastfm --disable-spotify --disable-verification

make clean all
```


Note that the project compiles an sqlite extension as a shared library. To avoid the shared library, we use "--disable-shared", and we apply the compile-time flag `-DSQLITE_CORE` (https://sqlite.org/loadext.html).


To run the server with the recompiled libraries, set the library path as follows:

```
export LD_LIBRARY_PATH=$WORKDIR/libwebsockets-2.0.3/build/lib/:$WORKDIR/libevent/build/lib/:$WORKDIR/pth-2.0.7/.libs/
```

To test the new version of the server:
```
./src/forked-daapd -d 5 -c ./forked-daapd.conf -f
```


To run the fuzzer:

```
afl-fuzz -d -i $WORKDIR/in-daapd/ -o $WORKDIR/out-daap/ -N tcp://127.0.0.1/3689 -P HTTP -D 300000 -m 1000 -t 3000+ -q 3 -s 3 -E -K -R ./forked-daapd -d 5 -c ./forked-daapd.conf -f
```

Note that the parameter `-D` is needed to let the server to initialize before fuzzing. The initialization takes few seconds.


