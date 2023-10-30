# dedupfs

The `AESEncryptionProvider` of `EncryptionFileSystem` is an encryption plugin with RocksDB. It depends on OpenSSL to show how an external plugin can bring its dependencies into the RocksDB build. It provides a factory function in a header file to show integration with RocksDB header includes. It can also be enabled in text-based options to demonstrate use of the static registration framework.

## Build

The code first needs to be linked under RocksDB's "plugin/" directory. In your RocksDB directory, run:

```
$ pushd ./plugin/
$ git clone https://github.com/acelyc111/encfs.git
```

Next, we can build and install RocksDB with this plugin as follows:

```
$ popd
$ make clean && DEBUG_LEVEL=0 ROCKSDB_PLUGINS="encfs" make -j48 db_bench install
```

Build by cmake and check the functionality:
```
#!/usr/bin/env bash

set -ex

# 1. build
#mkdir build && cd build
#cmake -DWITH_LZ4=1 -DCMAKE_BUILD_TYPE=Debug -DWITH_TESTS=1 -DROCKSDB_BUILD_SHARED=0 -DROCKSDB_PLUGINS=encfs ..

uris=("provider={id=AES;hex_instance_key=0123456789ABCDEF0123456789ABCDEF;method=AES128CTR};id=EncryptedFileSystem"
      "provider={id=AES;hex_instance_key=0123456789ABCDEF0123456789ABCDEF0123456789ABCDEF;method=AES192CTR};id=EncryptedFileSystem"
      "provider={id=AES;hex_instance_key=0123456789ABCDEF0123456789ABCDEF0123456789ABCDEF0123456789ABCDEF;method=AES256CTR};id=EncryptedFileSystem"
      "provider={id=AES;hex_instance_key=0123456789ABCDEF0123456789ABCDEF;method=SM4CTR};id=EncryptedFileSystem")
for uri in ${uris[*]}; do
  echo "${uri}"

  # Set env (select 1 of the 4)
  export TEST_FS_URI=${uri}

  # Run unit tests
  ./env_basic_test --gtest_filter=*CustomEnv*
  ./env_test --gtest_filter=CreateEnvTest.CreateEncryptedFileSystem

  # Run benchmarks
  ./db_bench --fs_uri="${uri}" --benchmarks="fillseq,readrandom,readseq" --compression_type=lz4 --num=1000000

  # Run ldb tools
  ./tools/ldb --fs_uri="${uri}" --db=/tmp/rocksdbtest-1000/dbbench/ put k v
  ./tools/ldb --fs_uri="${uri}" --db=/tmp/rocksdbtest-1000/dbbench/ get k
  ls -l /tmp/rocksdbtest-1000/dbbench | grep "log" | awk '{print $NF}' | xargs -i ./tools/ldb --fs_uri="${uri}" dump_wal --walfile=/tmp/rocksdbtest-1000/dbbench/{} | head
  ./tools/ldb --fs_uri="${uri}" --db=/tmp/rocksdbtest-1000/dbbench/ scan --hex | head
  ./tools/ldb --fs_uri="${uri}" --db=/tmp/rocksdbtest-1000/dbbench/ dump --hex | head
  ./tools/ldb --fs_uri="${uri}" --db=/tmp/rocksdbtest-1000/dbbench/ manifest_dump | head
  ./tools/ldb --fs_uri="${uri}" --db=/tmp/rocksdbtest-1000/dbbench/ list_live_files_metadata | head
done

# Unset the env
unset TEST_FS_URI
```

## Tool usage

For RocksDB binaries (such as the `db_bench` we built above), the plugin can be enabled through configuration. `db_bench` in particular takes a `--fs_uri` where we can specify "encfs" , which is the name registered by this plugin. Example usage:

```
$ ./db_bench --benchmarks=fillrandom --fs_uri="provider={id=AES;hex_instance_key=0123456789ABCDEF0123456789ABCDEF;method=AES128CTR};id=EncryptedFileSystem" --compression_type=none
```

## Application usage

The plugin's interface is also exposed to applications, which can enable it either through configuration or through code. Here is an example instantiating the plugin in code.

```
$ cat <<EOF >./tmp.cc
#include <rocksdb/plugin/encfs/encfs.h>

int main() {
  std::string fs_uri = "provider={id=AES;hex_instance_key=0123456789ABCDEF0123456789ABCDEF;method=AES128CTR}";
  Status s = Env::CreateFromUri(config_options, "", fs_uri, &raw_env, &env_guard)
  return 0;
}
EOF
$ g++ $(pkg-config --cflags rocksdb) -c ./tmp.cc -o ./tmp.o
$ g++ $(pkg-config --static --libs rocksdb) ./tmp.o -o ./tmp
$ ./tmp
```