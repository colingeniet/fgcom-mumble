.PHONY : all all-debug plugin libs test tools clean clean-all debug-on
CC=g++
CC_WIN=x86_64-w64-mingw32-g++-posix
CC_WIN32=i686-w64-mingw32-g++-posix
DEBUG=
THREADS_UNIX=-pthread
THREADS_WIN=-Wl,-Bstatic -lstdc++ -lpthread -lws2_32
override CFLAGS+=-O3 -I. -I./lib $(DEBUG)
override CFLAGS_WIN+=-static-libgcc -static-libstdc++ $(CFLAGS) -I./lib/openssl/include/
SSLFLAGS+=-lssl -lcrypto -DSSLFLAGS
SSLFLAGS_WIN+=-L./lib/openssl/ -static -lssl -lcrypto -DSSLFLAGS -lws2_32 -lcrypt32

# get version
GITCOMMIT:=$(shell git log -1 --pretty=format:"%h")
GITDATE:=$(shell git log -1 --pretty=format:"%cd" --date=short)
VERSION_V:=$(shell grep -m1 FGCOM_VERSION_MAJOR fgcom-mumble.h |grep -E -o '[0-9]+')
VERSION_M:=$(shell grep -m1 FGCOM_VERSION_MINOR fgcom-mumble.h |grep -E -o '[0-9]+')
VERSION_P:=$(shell grep -m1 FGCOM_VERSION_PATCH fgcom-mumble.h |grep -E -o '[0-9]+')
VERSION:=$(VERSION_V).$(VERSION_M).$(VERSION_P)

lib_OBJS := lib/radio_model.o lib/audio.o lib/io_plugin.o lib/io_UDPServer.o lib/io_UDPClient.o lib/garbage_collector.o

# Notice: Special calls
#
# `make DEBUG+="-DDEBUG"`                  enable debug code (use `make DEBUG+="-g3 -DDEBUG -Og"` for additional gdb symbols)
# `make CFLAGS+="-DNO_UPDATER" SSLFLAGS=`  skip auto-updater code
#

# FOR PRERELEASES: make always debug builds
#ifneq (,$(findstring 0,$(VERSION_V)))
#    DEBUG+=-g3 -DDEBUG
#endif

# If SSLFLAGS was set to empty, empty the win version too
ifeq ($(SSLFLAGS),)
    SSLFLAGS_WIN=
    override CFLAGS+=-DNO_UPDATER
    override CFLAGS_WIN+=-DNO_UPDATER
endif


# Compile all that stuff
all: plugin tools clean

# DEBUG MODE
#   convinience invocation for debug build fpr GDB
#   -g3:     gdb debugging symbols
#   -O0:     no optimizations, so we get better dbg output (but slows don code significantly)
#   -DDEBUG: makes debug code active (prints internal state to stdout every sec)
all-debug:
	make DEBUG+="-g3 -Og -DDEBUG" all

# make the plugin
plugin: libs
	$(CC) -shared -fPIC -o fgcom-mumble.so $(lib_OBJS) fgcom-mumble.cpp $(SSLFLAGS) $(CFLAGS) $(THREADS_UNIX)

# make all the libs
libs:  $(lib_OBJS)

%.o : %.cpp
	$(CC) -fPIC -c -o $@ $< $(CFLAGS) $(THREADS_UNIX)

# Compile testing tools
tools: libs
	$(CC) -o test/geotest lib/radio_model.o test/geotest.cpp $(CFLAGS)
	$(CC) -o test/frqtest lib/radio_model.o test/frqtest.cpp $(CFLAGS)


# catch2 unit tests linking against main
test: libs test/catch2/tests-main.o test/catch2/tests-main.o test/catch2/radioModelTest.o
	$(CC) -o test/catch2/radioModelTest.catch2 test/catch2/tests-main.o lib/radio_model.o test/catch2/radioModelTest.o $(CFLAGS) && test/catch2/radioModelTest.catch2
# ^ add more

# clean compile results
clean:
	rm -f *.o lib/*.o test/catch2/*.catch2

# clean compile results and binarys
clean-all: clean
	rm -f test/geotest test/frqtest
	rm -f fgcom-mumble*.so
	rm -f *.exe test/*.exe *.dll lib/*.dll
	rm -f test/catch2/*.o



# Build all win64 stuff
#   apt-get install mingw-w64
all-win: plugin-win64 tools-win64 clean

# build win64 test tools
tools-win64:
	$(CC_WIN) -o test/geotest.exe lib/radio_model.cpp test/geotest.cpp -static-libgcc -static-libstdc++ $(CFLAGS)
	$(CC_WIN) -o test/frqtest.exe lib/radio_model.cpp test/frqtest.cpp -static-libgcc -static-libstdc++ $(CFLAGS)

# build win64 plugin-dll and openssl
plugin-win64: openssl-win plugin-win64-only

plugin-win32: openssl-win32 plugin-win32-only

# just the windows plugin, no ssl in case we want to build repetively but want to avoid building openssl each time
plugin-win64-only:
	$(CC_WIN) -fPIC --shared -DMINGW_WIN64 -o fgcom-mumble.dll lib/io_plugin.cpp lib/radio_model.cpp lib/audio.cpp lib/io_UDPServer.cpp lib/io_UDPClient.cpp lib/garbage_collector.cpp fgcom-mumble.cpp $(SSLFLAGS_WIN) $(CFLAGS_WIN) $(THREADS_WIN)

plugin-win32-only:
	$(CC_WIN32) -m32 -fPIC --shared -DMINGW_WIN64 -o fgcom-mumble-x86_32.dll lib/io_plugin.cpp lib/radio_model.cpp lib/audio.cpp lib/io_UDPServer.cpp lib/io_UDPClient.cpp lib/garbage_collector.cpp fgcom-mumble.cpp $(SSLFLAGS_WIN) $(CFLAGS_WIN) $(THREADS_WIN)


# OpenSSL (static build)
# The sources are located under lib/openssl as git submodule, and supposed to point to the latest stable head
# Info on configure options: https://wiki.openssl.org/index.php/Compilation_and_Installation
openssl-win:
ifdef SSLFLAGS
	@echo "BUILD OpenSSL"
	git submodule init
	git submodule update
	cd lib/openssl/ && git reset --hard
	cd lib/openssl/ && ./Configure mingw64 -static no-weak-ssl-ciphers no-ssl2 no-ssl3 no-idea no-dtls1 --cross-compile-prefix=x86_64-w64-mingw32- && make clean && make
endif

openssl-win32:
ifdef SSLFLAGS
	@echo "BUILD OpenSSL"
	git submodule init
	git submodule update
	cd lib/openssl/ && git reset --hard
	cd lib/openssl/ && ./Configure mingw -static no-weak-ssl-ciphers no-ssl2 no-ssl3 no-idea no-dtls1 --cross-compile-prefix=i686-w64-mingw32- && make clean && make
endif


# Build release archives
release: clean-all test plugin plugin-win64 plugin-win32
	@echo GIT commit $(GITCOMMIT) from $(GITDATE), V$(VERSION)
	head -n 1 ../../README.md > releaseReadme.md
	@echo Version: $(VERSION) \($(GITCOMMIT) $(GITDATE)\) >> releaseReadme.md
	tail +2 ../../README.md >> releaseReadme.md
	
	# Linux + Windows combined
	mkdir fgcom-mumble-client-$(VERSION)
	mkdir fgcom-mumble-client-$(VERSION)/mumble-plugin
	rm -f fgcom-mumble-*.zip
	cp releaseReadme.md fgcom-mumble-client-$(VERSION)/README.md
	cp ../../LICENSE ../../README-de_DE.md ../../Readme.architecture.md ../plugin.spec.md fgcom-mumble-client-$(VERSION)/
	cp fgcom-mumble*.so  fgcom-mumble-client-$(VERSION)/mumble-plugin
	cp fgcom-mumble.dll fgcom-mumble-client-$(VERSION)/mumble-plugin
	cp fgcom-mumble-x86_32.dll fgcom-mumble-client-$(VERSION)/mumble-plugin
	cp fgcom-mumble.ini fgcom-mumble-client-$(VERSION)/mumble-plugin
	zip -r fgcom-mumble-client-$(VERSION).zip fgcom-mumble-client-$(VERSION)
	rm -rf fgcom-mumble-client-$(VERSION)
	
	# Just a binary package for the mumble updater
	# Note: this ZIP may has a special format
	zip fgcom-mumble-client-binOnly-$(VERSION).zip fgcom-mumble.so fgcom-mumble.dll fgcom-mumble-x86_32.dll
	
	rm releaseReadme.md
	@echo "\nPluginrelease $(VERSION) built successfully:"
	@ls -alh *$(VERSION)*
	@md5sum *$(VERSION)*

showVer:
	@echo "GITCOMMIT:$(GITCOMMIT)"
	@echo "GITDATE:$(GITDATE)"
	@echo "VERSION:$(VERSION)"
