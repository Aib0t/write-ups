# ProtoSSL

EAs in-house TLS solution. It’s not very clear why EAs opted for using their own ssl3 compatible library for it, but that’s how it is. A rather notable thing about it is a certificate check, so each title should be either patched or utilize a [bug](https://github.com/Aim4kill/Bug_OldProtoSSL) in ProtoSSL implementation. In the majority of titles the whole patch is a 1 byte change, so it’s not a big issue for the researchers.

The ProtoSSL bug described above affects FESL version 4+, while versions 2 and 3 require manual patching. In most cases a patch by **Luigi Auriemma** is sufficient for PC titles. You can grab it [here](https://aluigi.altervista.org/patches/fesl.lpatch) and the lpacher itself [here](https://aluigi.altervista.org/mytoolz/lpatch.zip). After that just patch the binary, and it will bypass the ssl check.

For a handful of PS3 titles a patch exists in the official RPCS3 repository. To check the list I use [patches list](https://github.com/illusion0001/rpcs3-game-patch/blob/main/patch.yml).
