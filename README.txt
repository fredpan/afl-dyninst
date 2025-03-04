American Fuzzy Lop + Dyninst == AFL Fuzzing blackbox binaries

The tool has two parts. The instrumentation tool and the instrumentation 
library. Instrumentation library has an initialization callback and basic 
block callback functions which are designed to emulate what AFL is doing
with afl-gcc/afl-g++/afl-as. 
Instrumentation tool (afl-dyninst) instruments the supplied binary by
inserting callbacks for each basic block and an initialization 
callback either at _init or at specified entry point.


Commandline options
-------------------

Usage: ./afl-dyninst-dfvD -i <binary> -o <binary> -l <library> -e <address> -E <address> -s <number> -S <funcname> -m <size>
   -i: input binary 
   -o: output binary
   -d: do not instrument the binary, only supplied libraries
   -l: linked library to instrument (repeat for more than one)
   -r: runtime library to instrument (path to, repeat for more than one)
   -e: entry point address to patch (required for stripped binaries)
   -E: exit point - force exit(0) at this address (repeat for more than one)
   -s: number of initial basic blocks to skip in binary
   -m: minimum size of a basic bock to instrument (default: 1)
   -f: try to fix a dyninst bug that leads to crashes
   -S: do not instrument this function (repeat for more than one)
   -D: instrument fork server and forced exit functions but no basic blocks
   -v: verbose output

Switch -l is used to supply the names of the libraries that should 
be instrumented along the binary. Instrumented libraries will be copied
to the current working directory. This option can be repeated as many times
as needed. Depending on the environment, the LD_LIBRARY_PATH should be set 
to point to instrumented libraries while fuzzing. 

Switch -e is used to manualy specify the entry point where initialization
callback is to be inserted. For unstipped binaries, afl-dyninst defaults 
to using _init of the binary as an entry point. In case of stripped binaries
this option is required and is best set to the address of main which 
can easily be determined by disassembling the binary and looking for an 
argument to __libc_start_main. 

Switch -E is used to specify addresses that should force a clean exit
when reached. This can speed up the fuzzing tremendously.

Switch -s instructs afl-dyninst to skip the first <number> of basic
blocks. Currently, it is used to work around a bug in Dyninst
but doubles as an optimization option, as skipping the basic blocks 
of the initialization rutines makes things run faster. If the instrumented
binary is crashing by itself, try skiping a number of blocks.

Switch -r allows you to specify a path to the library that is loaded
via dlopen() at runtime. Instrumented runtime libraries will be 
written to the same location with a ".ins" suffix as not to overwrite
the original ones. Make sure to backup the originals and then rename the
instrumented ones to original name. 

Switch -m allows you to only instrument basic blocks of a minimum size - the
default minimum size is 1

Switch -f fixes a dyninst bug that lead to bugs in the instrumented program:
our basic block instrumentation function loaded into the instrumentd binaries
uses the edi/rdi. However dyninst does not always saves and restores it when
instrumenting that function leading to crashes and changed program behaviour
when the register is used for function parameters.

Switch -S allows you to not instrument specific functions.
This options is mainly to hunt down bugs in dyninst.

Switch -D installs the afl fork server and forced exit functions but no
basic block instrumentation. That would serve no purpose - unless there is
another interesting tool coming up ... :)


Compiling:
----------

1. Edit the Makefile and set DYNINST_ROOT and AFL_ROOT to appropriate paths. 
2. make
3. make install


Example of running the tool
---------------------------

Dyninst requires DYNINSTAPI_RT_LIB environment variable to point to the location
of libdyninstAPI_RT.so.

$ export DYNINSTAPI_RT_LIB=/usr/local/lib/libdyninstAPI_RT.so
$ ./afl-dyninst -i ./rar -o ./rar_ins -e 0x4034c0 -s 100
Skipping library: libAflDyninst.so
Instrumenting module: DEFAULT_MODULE
Inserting init callback.
Saving the instrumented binary to ./rar_ins...
All done! Happy fuzzing!

Here we are instrumenting  the rar binary with entrypoint at 0x4034c0
(manualy found address of main), skipping the first 100 basic blocks 
and outputing to rar_ins. 


Running AFL on instrumented binary
----------------------------------

NOTE: The instrumentation library "libDyninst.so" must be available in the current working
directory or LD_LIBRARY_PATH as that is where the instrumented binary will be looking for it.

Since AFL checks if the binary has been instrumented by afl-gcc,AFL_SKIP_BIN_CHECK environment 
variable needs to be set. No modifications to AFL it self is needed. 
$ export AFL_SKIP_BIN_CHECK=1
Then, AFL can be run as usual:
$ afl-fuzz -i testcases/archives/common/gzip/ -o test_gzip -- ./gzip_ins -d -c 

Note that there are the helper scripts afl-fuzz-dyninst.sh and afl-dyninst.sh for you which set the
required environment variables for you.


=================================
Below config specific for UofT ECE 1776, 2019:

# Install spack:

   $ cd /home/ubuntu
   $ sudo mkdir spack
   $ cd spack
   $ git clone https://github.com/spack/spack.git

   
# Install Dyninst from spack

   $ cd spack/bin
   $ ./spack install dyninst


# Configure dyninst

   $ export LD_LIBRARY_PATH=/usr/local/lib:/home/ubuntu/spack/opt/spack/linux-ubuntu16.04-x86_64/gcc-5.4.0/dyninst-10.1.0-unvujmg7dbajzysqjuedjqbvgz2izc4u/lib/


# Install $ config libiberty

   $ sudo apt-get install libiberty-all-dev
   $ dpkg -L libiberty-dev | grep -F iberty.a


# Install afl-dyninst

   $ cd ~
   $ sudo mkdir afl-dyninst
   $ git clone https://github.com/fredpan/afl-dyninst.git
   $ vim Makefile
   Change DYNINST_ROOT and AFL_ROOT of the Makefile (the file name could be a little bit different):
   $ Makefile = /home/ubuntu/spack/opt/spack/linux-ubuntu16.04-x86_64/gcc-5.4.0/dyninst-10.1.0-unvujmg7dbajzysqjuedjqbvgz2izc4u
   $ AFL_ROOT = /home/ubuntu/shellphish-afl/shellphish_afl/../bin/afl-unix
   $ make
   $ make install


# Config the afl-dyninst

   $ export DYNINSTAPI_RT_LIB=/home/ubuntu/spack/opt/spack/linux-ubuntu16.04-x86_64/gcc-5.4.0/dyninst-10.1.0-unvujmg7dbajzysqjuedjqbvgz2izc4u/lib/libdyninstAPI_RT.so

  
# Build the AFL Binaries (Problem here: Assertion error and/or copy_states not enabled):

   ./afl-dyninst -i /home/ubuntu/fuzzer/objdump -o ./fredpan -d

