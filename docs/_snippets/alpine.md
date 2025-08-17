# Alpine Linux
Various snippets that work with Alpine Linux.

## Licences of installed packages
```sh
cat /lib/apk/db/installed | grep 'L:' | sort | uniq -c | sort -n
```

### Example
```
/ # cat /lib/apk/db/installed | grep 'L:' | sort | uniq -c | sort -n
      1 L:MIT AND BSD-2-Clause AND GPL-2.0-or-later
      1 L:MPL-2.0 AND MIT
      1 L:Zlib
      2 L:Apache-2.0
      3 L:MIT
      7 L:GPL-2.0-only
```
