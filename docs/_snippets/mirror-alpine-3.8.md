Snippet for mirroring alpine 3.8 packages - essentially x86_64.

If you are going to run it be sure to consider a different mirror, there may
be one closer to you.

For [rsync][1] on Windows, I use [cwRsync][2], click Free Rsync Client then
[download][3] to find the download (at least as of 2022-10-30).

```
SET source=rsync://mirror.aarnet.edu.au/pub/alpine/v3.8
SET dest=D:/Downloads/Linux/alpine
cd D:
cd %dest%
D:\Programs\rsync\rsync.exe --verbose ^
       --archive ^
        --no-motd ^
        --update ^
        --hard-links ^
        --delete ^
        --delete-after ^
        --delay-updates ^
        --timeout=600 ^
        --exclude x86 --exclude aarch64 --exclude armhf  --exclude armv7 ^
        --exclude ppc64le --exclude s390x --exclude aarch64 ^
        --exclude releases ^
    %SOURCE% .
```

[1]: https://rsync.samba.org/
[2]: https://www.itefix.net/cwrsync
[3]: https://itefix.net/dl/free-software/cwrsync_6.2.7_x64_free.zip
