vita-toolchain
==============
These are a set of tools that, along with an ARM compiler and linker, provides a
build system for PlayStation Vita(TM). Look at
[buildscripts](https://github.com/vitasdk/buildscripts) for more information on
building the complete toolchain with binutils/gcc/newlib. As such, this repo
only contains Vita specific host tools. Check out the
[specifications](doc/specifications.pdf) for more details on the Vita executable
format, these tools, and their inputs/outputs (including the YAML configuration 
format).

## vita-toolchain file formats
|name|processor|description|
|-----|-----|-----|
|elf|-|A normal .elf format|
|velf|vita-elf-create.c|The vita's elf format (elf + some module infos)|
|self|-|The signed velf file + self headers|
|fself|vita-make-fself.c|The self file. but not signed. (fake self)|
|nid_db_json|-|really old nid db format|
|nid_db_classic|vita-import-parse.c <br> vita-nid-db-yml.c|Base of the most used formats currently|
|nid_db_classic_v2|vita-import-parse.c <br> vita-nid-db-yml.c|Added library `version` key to nid_db_classic|
|nid_db_classic_v3|vita-nid-db-yml.c|Added library `stubname` key to nid_db_classic_v2|
|nid_db_bypass|vita-nid-bypass.c|The yml format to bypass duplicate entry names in vita-libs-gen-2|
|module_config|vita-export-parse.c|The yml format for more detailed module configuration|

### vita-elf-create
```
usage: vita-elf-create [-v|vv|vvv] [-n] [-e config.yml] input.elf output.velf
    -v,-vv,-vvv:    logging verbosity (more v is more verbose)
    -n         :    allow empty imports
    -e yml     :    optional config options
    input.elf  :    input ARM ET_EXEC type ELF
    output.velf:    output ET_SCE_RELEXEC type ELF
```
Converts a standard `ET_EXEC` ELF (outputted by `arm-vita-eabi-gcc` for example)
to the Sony ELF format.

vita-elf-create also adds special symbols defined programmatically to module info.

|type|mode|name|prototype|used by|
|-----|-----|-----|-----|-----|
|variable|C|module_sdk_version|const SceUInt32 module_sdk_version;|Kernel|
|variable|C|sceUserMainThreadName|const char sceUserMainThreadName[];|Kernel|
|variable|C|sceUserMainThreadPriority|const SceUInt32 sceUserMainThreadPriority;|Kernel|
|variable|C|sceUserMainThreadStackSize|const SceSize sceUserMainThreadStackSize;|Kernel|
|variable|C|sceUserMainThreadCpuAffinityMask|const SceUInt32 sceUserMainThreadCpuAffinityMask;|Kernel|
|variable|C|sceUserMainThreadAttribute|const SceUInt32 sceUserMainThreadAttribute;|Kernel|
|variable|C|sceKernelPreloadModuleInhibit|const SceUInt32 sceKernelPreloadModuleInhibit;|Kernel|
|variable|C|sceLibcHeapUnitSize1MiB|unknown|SceLibc|
|variable|C|sceLibcHeapSize|SceSize sceLibcHeapSize;|SceLibc|
|variable|C|sceLibcHeapInitialSize|unknown|SceLibc|
|variable|C|sceLibcHeapExtendedAlloc|unknown|SceLibc|
|variable|C|sceLibcHeapDelayedAlloc|unknown|SceLibc|
|variable|C|sceLibcHeapDetectOverrun|unknown|SceLibc|
|function|C|user_malloc_init|unknown|SceLibc|
|function|C|user_malloc_finalize|unknown|SceLibc|
|function|C|user_malloc|unknown|SceLibc|
|function|C|user_free|unknown|SceLibc|
|function|C|user_calloc|unknown|SceLibc|
|function|C|user_realloc|unknown|SceLibc|
|function|C|user_memalign|unknown|SceLibc|
|function|C|user_reallocalign|unknown|SceLibc|
|function|C|user_malloc_stats|unknown|SceLibc|
|function|C|user_malloc_stats_fast|unknown|SceLibc|
|function|C|user_malloc_usable_size|unknown|SceLibc|
|function|C|user_malloc_for_tls_init|unknown|SceLibc|
|function|C|user_malloc_for_tls_finalize|unknown|SceLibc|
|function|C|user_malloc_for_tls|unknown|SceLibc|
|function|C|user_free_for_tls|unknown|SceLibc|
|function|C++|_Z8user_newj|user_new(std::size_t) throw(std::badalloc)|SceLibc|
|function|C++|_Z8user_newjRKSt9nothrow_t|user_new(std::size_t, std::nothrow_t const&)|SceLibc|
|function|C++|_Z14user_new_arrayj|user_new_array(std::size_t) throw(std::badalloc)|SceLibc|
|function|C++|_Z14user_new_arrayjRKSt9nothrow_t|user_new_array(std::size_t, std::nothrow_t const&)|SceLibc|
|function|C++|_Z11user_deletePv|user_delete(void*)|SceLibc|
|function|C++|_Z11user_deletePvRKSt9nothrow_t|user_delete(void*, std::nothrow_t const&)|SceLibc|
|function|C++|_Z17user_delete_arrayPv|user_delete_array(void*)|SceLibc|
|function|C++|_Z17user_delete_arrayPvRKSt9nothrow_t|user_delete_array(void*, std::nothrow_t const&)|SceLibc|

### vita-elf-export
```
usage: vita-elf-export mod-type elf exports imports
    mod-type: valid values: 'u'/'user' for user mode, else 'k'/'kernel' for kernel mode
    elf: path to the elf produced by the toolchain to be used by vita-elf-create
    exports: path to the config yaml file specifying the module information and exports
    imports: path to write the import yaml generated by this tool
```
Creates an import database YAML from a Sony ELF and config YAML. This import
database is a YAML file (not to be confused with the config YAML file) which
defines the NID mappings for a given module. This can be used by other tools for
debugging and reversing purposes.

### vita-libs-gen
```
usage: vita-libs-gen nids.yml [extra.yml ...] output-dir
    -c: Generate CMakeLists.txt instead of a Makefile
```
Given a list of import database YAML files (either from `vita-elf-create` or
manually written by hackers for Sony owned modules), this will generate stub
libraries that can be linked to such that `vita-elf-create` can properly
generate a Sony ELF. After calling `vita-libs-gen` you need to run `make` or 
`cmake` in the output directory to build the stubs.

vita-libs-gen supports nid_db_classic_v2 yml file

Warning: You cannot add more than 1024 entries to a single library in cmake mode.

### vita-libs-gen-2
```
usage: vita-libs-gen-2 -yml=<nids_db.yml|nids_db_yml_dir> -output=<output_dir> [-cmake=<true|false>] [-ignore-stubname=<true|false>]
```
Enhanced version of vita-libs-gen.
- better cleeanup for make version
- no 1024 limite for entries number in cmake mode
- support multi yml in once
- support stubname mapping

vita-libs-gen-2 supports nid_db_classic_v3 yml file

### vita-nid-check
```
usage: vita-nid-check -dbdirver=<./path/to/db_dir> [-bypass=<./path/to/bypass.yml>] [-dbg=<debug|trace>] 
```

Checks all files in the selected directory.
- Is yml file
- Is entry sorted
- Is no duplicated name

Also cannot contain multiple firmware versions within the selected directory.

vita-nid-check supports nid_db_classic_v3 yml file

### vita-make-fself
```
usage: vita-make-fself [-s|-ss] [-c] input.velf output-eboot.bin
    -s : Generate a safe eboot.bin. A safe eboot.bin does not have access
    to restricted APIs and important parts of the filesystem.
    -ss: Generate a secret-safe eboot.bin. Do not use this option if you don't know what it does.
    -c : Enable compression.
```
Generates a FSELF (the format expected of `eboot.bin` loaded by Vita) which
wraps around the Sony ELF file. Optionally supports compression of the input
ELF. Also allows marking a homebrew as "safe", which prevents it from harming
the system.

### vita-mksfoex
```
usage: mksfoex [options] TITLE output.sfo
    -d NAME=VALUE   Add a new DWORD value
    -s NAME=STR     Add a new string value
````
Generates a `param.sfo` file for a given title. `param.sfo` is required to
launch a homebrew from LiveArea. The `TITLE` id can be anything but it is
recommended that you use `XXXXYYYYY` where `XXXX` is an author specific
identifier and `YYYYY` is a unique number identifying your homebrew. For
example, molecularShell uses `MLCL00001`.

### vita-pack-vpk
```
Usage:
    vita-pack-vpk [OPTIONS] output.vpk

  -s, --sfo=param.sfo     sets the param.sfo file
  -b, --eboot=eboot.bin   sets the eboot.bin file
  -a, --add src=dst       adds the file src to the vpk as dst
  -h, --help              displays this help and exit
```
Generates a VPK homebrew package. `eboot.bin` and `param.sfo` are required.

## Development
Required libraries are 
[libelf](http://www.mr511.de/software/libelf-0.8.13.tar.gz), 
[zlib](http://zlib.net/zlib-1.2.8.tar.gz), 
[libzip](https://nih.at/libzip/libzip-1.1.3.tar.gz), and 
[libyaml](http://pyyaml.org/download/libyaml/yaml-0.1.7.tar.gz). Please note 
that there are some compatibility problems with built-in libelf so it is 
recommended that you download it from the provided link.

After getting the dependencies, build with
```
mkdir build && cd build
cmake ..
make
```

### Note on Naming
Early in the development, there was a confusion on the meaning of "module" and
"library" in context of the Vita. After the tools were written initially, we
decided to reverse the meaning of those two words. All user-facing usage of
those words have been changed (console outputs, messages, etc) but it is too
much word to change all internal usage of those words (function/variable names,
etc). Therefore you may be confused when reading the source since the meaning of
"module" and "library" is used inconsistently. It would be great if someone
could take the time to correct all the usages ("module" exports one or more
"libraries" and imports zero or more "libraries"; `eboot.bin` is a module).
