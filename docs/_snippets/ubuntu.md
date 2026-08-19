# Ubuntu
Various snippets that work with Ubuntu.

## Replace mirror

* This replaces the default mirror with one from Australia (my home country).
* Mostly this is intended for running on containers but can also be handy for
  VMs.

### sources.list
```sh
sed -i 's|http://archive\.ubuntu\.com/ubuntu/|https://mirror.aarnet.edu.au/pub/ubuntu/archive/|g' /etc/apt/sources.list
```

## DEB822

[Debian RFC822 control data format](https://man7.org/linux/man-pages/man5/deb822.5.html)

```sh
sed -i 's|http://archive\.ubuntu\.com/ubuntu/|https://mirror.aarnet.edu.au/pub/ubuntu/archive/|g' /etc/apt/sources.list.d/ubuntu.sources
```
