---
title:  "How to Install New Node.js with Old glibc without sudo"
mathjax: true
---

All the story began from trying to use [Copilot with Neovim](https://github.com/github/copilot.vim) on an old computing cluster where the glibc version is quite old.
To use Copilot with Neovim, we need to have [Node.js >= 18.x](https://nodejs.org/en), and Node.js >= 18.x demands glibc >=2.28.

First, I downloaded [Node.js 18.20.6](https://nodejs.org/download/release/v18.20.6/node-v18.20.6-linux-x64.tar.gz) which is a pre-built version, which means we can directly run it after unzipping the tar ball.
```shell
wget https://nodejs.org/download/release/v18.20.6/node-v18.20.6-linux-x64.tar.gz
tar zxvf node-v18.20.6-linux-x64.tar.gz
cd node-v18.20.6-linux-x64/bin
./node --version
```
However, on my old computing cluster, the glibc is too old.
Therefore, if I try to run `./node --version` at `node-v18.20.5-linux-x64/bin`, I get
```txt
./node: /lib64/libm.so.6: version `GLIBC_2.27' not found (required by ./node)
./node: /lib64/libc.so.6: version `GLIBC_2.25' not found (required by ./node)
./node: /lib64/libc.so.6: version `GLIBC_2.28' not found (required by ./node)
```

As a normal user of the computing cluster, I obviously don't have the sudo privilege, but I could compile a newer glibc in my local directory.
Again, I downloaded a newer glibc, specifically [glibc 2.38](http://ftp.gnu.org/gnu/libc/glibc-2.38.tar.gz), unzipped it and compile it to a local directory that I can write to.
```shell
tar xvzf glibc-2.38.tar.gz
cd glibc-2.38
mkdir build
cd build
../configure --prefix=$HOME/local/glibc-2.38
make -j$(nproc)
make install
```

Some people might believe that if they add the newer glibc into the environment variable, `LD_LIBRARY_PATH`, everything will be fine and so will Node.js and Copilot with Neovim.
However, I found that if I do
```shell
## Don't do this in your .bashrc or .zshrc!!!!
## Don't try to replace your old glibc with the newer version
## Otherwise, you cannot change it back easily if you don't have other terminals available without this change.
export LD_LIBRARY_PATH=$HOME/local/glibc-2.38/lib:$LD_LIBRARY_PATH
```
I couldn't even `ls` a directory but got `Segmentation Fault`.

To solve this issue, the best solution I found is to call `$HOME/local/glibc-2.38/lib/ld-linux-x86-64.so.2 /path/to/Node.js/bin/node` instead of calling `/path/to/Node.js/bin/node` directly.
Specifically, here is what I did
```shell
mkdir -p $HOME/local/bin
cd $HOME/local/bin
ln -s /path/to/Node.js/bin/* .  ## link all the commands to this new directory
rm node  ## remove node, because we'd like to change it
nvim node ## Open and edit a new file named node
```

In this new file, I wrote
```shell
#!/bin/bash

$HOME/local/glibc-2.38/lib/ld-linux-x86-64.so.2 /path/to/Node.js/bin/node "$@"
```
Save it and exit Neovim. Make sure that `$HOME/local/bin` is in the front of your `PATH`.
You may check it by `which node`, and got `$HOME/local/bin/node`.

Now let's try `node --version`, and I got `v18.20.5`. Brilliant, we finally got Node.js 18.x work and Copilot should also work in Neovim now.
How it works? Well, I am not hundred percent sure, but the essential principle would be every time we call `ld-linux-x86-64.so.2 node`, the newer glibc is applied for linked for `node`, so that demand of having newer version of glibc is fulfilled.

