---
layout: post
title:  "Minix meets fsspec and P9 protocol"
date:   2025-12-26 14:30:00 +1030
---

Expanding on the [previous post][0] about Minix file system, a couple of extra
operations are now supported and the file system is exposed to [`fsspec`][1]
and  [`P9` file system protocol][2].

Started off with adding a [`walk`][3] function to walk over the entire file
system. The first step here was to implement a version of `scandir`. The
unfortunate thing about that is the `os.DirEntry` type that the `os.scandir()`
function returns can not be constructed by user-code, so need our own type for
that. The implementation was fairly straight forward, as its just a matter
of using the existing `read_directory` function from last time but converting
the results to the `DirEntry` excluding the parent (`..`) and current
directory (`.`) items.

## Walk
A directory tree generator, whose implementation is very similar to [`walk`][3]
because it is essentially based on it. The thing that was very tempting is to
change the API so it returned the `DirEntry` items rather than only the name.

The `topdown`argument isn't supported and instead in the behaviour is defaulted
to `True`. The main difference here is it iterates of each sub-directory
and yields its result instead of yielding the information for the current
directory.

The `onerror` argument isn't supported which helps simply this the `os.walk`
needs to manually do the iteration by calling `next()` to account for errors
raised when querying the next item in the directory.

```python
def walk(path: os.PathLike | str, system: LoadedSystem):
    path = pathlib.PurePosixPath(os.fspath(path))

    directories = []
    non_directories = []

    directory_iterator = system.scandir(path)
    for entry in directory_iterator:
        if entry.is_dir():
            directories.append(entry.name)
        else:
            non_directories.append(entry.name)

    # This assumes top-down.
    yield path, directories, non_directories

    for directory in directories:
        yield from walk(path / directory, system)
```

## fsspec
The [`fsspec`][6] package provides a more common interface over a higher level
file system interface. Think of it as `pathlib` but for the file system.

That said, `Path` feels like its doing more than just one thing as it has
functions for interrogating hte path, such as `stat()` and querying if it its
file, including being able to rename, create/remove directories, iterate over
directories. As such, implementing `Path` would get you most the way there.

A function it implements is `disk_usage()` which tallies the file size of
each file and/or provides mapping for each path and its size.

Ultimately it should bea  unified interface for file systems such that you
can write your functions to use the fsspec interface and don't need to add
special handling based on what time of system it is. Typically used for
applications that separate compute from data storage.

This turns out to be quite straight forward with `fsspec.AbstractFileSystem`,
at least just for dealing with folders (i.e. no reading), as the only
function that needs to be implement is `ls()` and the `root_marker` property
needed to be set to define how to refer to paths as absolute paths.

```python
class MinixFileSystem(fsspec.AbstractFileSystem):
    """A backend for a disk image formatted as the Minix file system format."""

    root_marker = "/"

    def __init__(self, image_path: pathlib.Path, *args, **kwargs):
        super().__init__(args, kwargs)
        self.reader = image_path.open("rb")
        self.system = minix.LoadedSystem.from_reader(self.reader)

    def ls(self, path, detail=True):

        if detail:
            entries = self.system.scandir(path)
            return sorted(
                (
                    {
                        # As per the documentation of base class, this must be
                        # the full path to the entry without protocol.
                        "name": str(entry.path),
                        "size": entry.stat().st_size,
                        "type": "directory" if entry.is_dir() else "file",
                    }
                    for entry in entries
                ),
                key=operator.itemgetter("name"),
            )

        inode = self.system._path_to_inode(pathlib.PurePosixPath(path))
        children = self.system.read_directory(inode).entries
        return sorted(child.filename.decode("utf-8") for child in children)
```

The `TarFileSystem` implementation simply leaves closing the open file to the
garbage collector, so the class above does the same.

The `info()` function was overwritten simply to provide an optimisation here.
That said, it probably be simply to cache the last call to `scandir()` and
reuse that if it the same path.

```python
    def info(self, path):
        """Give details of entry at path."""
        # This avoids the behave for the base class where it queries the
        # children of the parent folder (i.e. it performs a ls() first).
        inode = self.system._path_to_inode(pathlib.PurePosixPath(path))
        if inode.is_directory:
            return {"name": path, "size": 0, "type": "directory"}
        return {"name": path, "size": inode.file_size, "type": "file"}
```

### Usage

```python
def example():
    fs = MinixFileSystem(pathlib.Path.cwd() / "minixfs.raw")
    for entry in fs.ls("/", detail=False):
        print(entry)

    for entry in fs.ls("/", detail=True):
        print(entry)

    print(fs.exists("/users/ast/welcome"))
    print(fs.exists("/users/ast/books"))
    print(fs.exists("/users/ast"))

    print("Is file", "Is directory")
    print(fs.isfile("/users/ast/welcome"), fs.isdir("/users/ast/welcome"))
    print(fs.isfile("/users/ast/books"), fs.isdir("/users/ast/books"))
    print(fs.isfile("/users/ast"), fs.isdir("/users/ast"))

    print(fs.stat("/users/ast/welcome"))
    print(fs.stat("/users/ast/welcome"))
    print(fs.stat("/users/ast/books"))

    print("Testing fs.walk()")
    for _, dirs, files in fs.walk("/users", detail=False):
        if files:
            print(files)
    print()

    print("Testing fs.walk('/')")
    for _, dirs, files in fs.walk("/", detail=False):
        if files:
            print(files)
    print()

    assert fs.isdir("/users")
    print("find('/users', detail=True)", fs.find("/users", detail=True))
    print("find('/users')", fs.find("/users"))
    print("Disk usage of /users", fs.disk_usage("/users", total=False))
    print("Disk usage of /users total", fs.disk_usage("/users"))
    assert fs.isdir("/")
    print("Root information", fs.info("/"))
    print("Disk usage of /", fs.disk_usage("/"))
```

### Output

```
.
..
media
system
users
{'name': '/media', 'size': 128, 'type': 'directory'}
{'name': '/system', 'size': 192, 'type': 'directory'}
{'name': '/users', 'size': 192, 'type': 'directory'}
True
True
True
Is file Is directory
True False
True False
False True
{'name': '/users/ast/welcome', 'size': 6, 'type': 'file'}
{'name': '/users/ast/welcome', 'size': 6, 'type': 'file'}
{'name': '/users/ast/books', 'size': 53, 'type': 'file'}
Testing fs.walk()
['books', 'welcome']

Testing fs.walk('/')
['fur-elise.mid']
['big-buck-bunny', 'kv']
['ls', 'ping', 'rm', 'sh', 'tar']
['books', 'welcome']

find('/users', detail=True) {'/users/ast/books': {'name': '/users/ast/books', 'size': 53, 'type': 'file'}, '/users/ast/welcome': {'name': '/users/ast/welcome', 'size': 6, 'type': 'file'}}
find('/users') ['/users/ast/books', '/users/ast/welcome']
Disk usage of /users {'/users/ast/books': 53, '/users/ast/welcome': 6}
Disk usage of /users total 59
Root information {'name': '/', 'size': 0, 'type': 'directory'}
Disk usage of / 59
```

### Next Time
Handle the reading of a file.
```python
def read_example():
    fs = MinixFileSystem(pathlib.Path.cwd() / "minixfs.raw")
    welcome = fs.cat_file("/users/ast/welcome")
    print(welcome)

    welcome = fs.cat("/users/ast/welcome")
    print(welcome)
```

This involves exposing the `_fetch_range()` function or ovewriting the `_open`
function.

## p9 file system
The next experiment was to write a module which exposes the Minix file system
image using the 9p remote filesystem protocol from Plan 9. The Linux kernel
has an implementations of this protocol called [v9fs][5], which allows
mounting a file system served over this protocol.

For this project, the `pyfs-py` package by pbchekin will be used. Originally,
the implementation for `pyroute2` looked very promising as it supports async
interface however, it has the downside of having too many other features and
lots of the code is Linux specific where the former package can work on
Microsoft Windows.

### Implementation

Starting off basing it on the example server implementation where instead of
constructing a LocalFileSystem, to expose a directory on the hosting machine,
it constructs our `MinixFs` class and mounts that as the file system.

```python
class MinixFs:
    # Class to implement the interface needed by the py9p.Server class.
    pass

if __name__ == "__main__":
    srv = py9p.Server(listen=("0.0.0.0", 8999), chatty=False)
    if not srv.chatty:
        logging.basicConfig(level=logging.INFO)
    with MinixFs(pathlib.Path.cwd() / "minixfs.raw") as fs:
        srv.mount(fs)
        if not srv.chatty:
            print(f"Listening {srv.host} on port {srv.port}")
        srv.serve()
```

This will fail as we haven't implemented the expected functions.

The key aspect of this is being able to convert a path to a `py9p.Dir` object
which is more of a `stat`-like call. In the end, the function for handling this
took a `os.stat_result` which the `minix` package could produce and create
the `py9p.Dir` from it.


Required implementation:

* The file system class should have a `root` property of type `py9p.Dir`, which
  represents the root. This would use the function mentioned above to query
  the file status of the root file and convert it ot the `Dir` object.
* `pathtodir(self, path) -> py9p.Dir` - Given the path returns the Dir object
   representing that path,. As mentioned above this is essentially `stat()` of
   the path and convert it to the resulting type.
* `walk(self, srv, req)` - Descend a directory hierarchy
* `stat(self, srv, req)` - Inquire about file attributes.\
  Essentially, this simply calls `pathtodir` on the path and adds it to the
  result.
* `open(self, srv, req)` - Prepare a fid for I/O on an existing file.
    * For a directory, this essentially does nothing
    * For a file, this can open the file and keep track of it later.
* `read(self, srv, req)` - Prepare a fid for I/O on an existing file.
    *  For a directory, this reads the contents of the directory, so similar
      to `scandir()` but does `pathtodir()` on each child and includes that
      in the results.
    * For a file, it seeks to the offset and reads the given bytes.

As this was being implemented, I originally didn't have the `open()` function
for the Minix file system written so I simply had no file access, meaning
`cat` on a file didn't work but I could browse the file system and `find`
worked.

### Reading a file

Since I really wanted to be able to `cat` the file, I ended up going back
and implementing `open()` and introducing a file-like object to the `minix`
module called `FileIO` which was able to read and had basic seek capabilities.
Other than implementing the `typing.BinaryIO` interface which was fairly
basic for the properties themselves, it simply required refactoring the
existing read function to work from this class.

Since most the heavy lifting was no in FileIO, this was what the `LoadedSystem`
ended up.
```python
class LoadedSystem
   ...
    def open(self, path: os.PathLike | str, mode: str) -> FileIO:
        """Open file and return a stream.  Raise OSError upon failure."""
        inode = self._path_to_inode(pathlib.PurePosixPath(path))
        return FileIO(path, mode, inode, self)

    def read(self, path: pathlib.PurePosixPath | str) -> bytes:
        """Read the contents of a file at the given path."""
        with self.open(path, "rb") as readable_file:
            return readable_file.read()
```

Putting the above together:
```python
class MinixFs:
    ...

    def open(self, srv, req):
        ...

        if not (f.qid.type & py9p.QTDIR) and not stat_module.S_ISLNK(stat.st_mode):
            f.fd = self.system.open(f.localpath, m)

        srv.respond(req, None)

    def read(self, srv, req):
        f = self.getfile(req.fid.qid.path)
        if not f:
            srv.respond(req, "unknown file")
            return

        inode = self.system._path_to_inode(f.localpath)
        if f.qid.type & py9p.QTDIR:
            LOGGER.info("read directory: %s", f.localpath)
            directory = self.system.read_directory(inode)
            l = filter(
                lambda entry: entry.filename not in (b".", b".."), directory.entries
            )
            req.ofcall.stat = [
                self.pathtodir(f.localpath / entry.filename.decode("utf-8"))
                for entry in l
            ]
        else:
            LOGGER.info("read file: %s", f.localpath)
            f.fd.seek(req.ifcall.offset)
            req.ofcall.data = f.fd.read(req.ifcall.count)
        srv.respond(req, None)
```

This includes the directory implementation of read that was overlooked above,
it now able to cat.

### Usage
* Start the server: `uv run minix_p9fs`
* From Linux
    * `mount -t 9p -o trans=tcp  172.19.16.1 -o port=8999 /media/minix/`
    * ```
    $ ls /media/minix/
media   system  users
    $ ls /media/minix/users
ast   dick  erik  jim
    $ find /media/minix
/media/minix/
/media/minix/system
/media/minix/system/bin
/media/minix/system/bin/sh
/media/minix/system/bin/ls
/media/minix/system/bin/rm
/media/minix/system/bin/ping
/media/minix/system/bin/tar
/media/minix/system/lib
/media/minix/system/etc
/media/minix/system/var
/media/minix/users
/media/minix/users/dick
/media/minix/users/erik
/media/minix/users/jim
/media/minix/users/ast
/media/minix/users/ast/welcome
/media/minix/users/ast/books
/media/minix/media
/media/minix/media/audio
/media/minix/media/audio/fur-elise.mid
/media/minix/media/videos
/media/minix/media/videos/big-buck-bunny
/media/minix/media/videos/kv
    $ cat /media/minix/users/ast/books
OS: Design and Implementation
To Kill a Mocking Bird
    ```
    * `umount /media/minix`

### Wireshark

This is a brief look at the protocol itself, which since the [p9fs-py][6]
package was used there wasn't a need to dive deep into the protocol.

Messages sent from when
 `mount -t 9p -o trans=tcp  172.19.16.1 -o port=8999 /media/minix/` is called
 to when it completes.

* Client sends `Tversion` message with the version being `9P2000.L` in this
  case.
* Server replies with a `Rversion` message with `9P2000`.
* Client sends a `Tattach` message (messages to establish a connection).
    * Uname: nobody
    * Aname: <empty>
* Server replies with `Rattach` with the `Qid` of the root object.
* Client sends a `Tstat` and server replies with `Rstat` for root object.

If the file system provided has a `attach` class method then that will be
called when the `Tattach` is called. otherwise it responds with teh root i>D

There were several messages sent when doing `ls /media/minix`.

* Multiple stat calls
* Walk
* Open
* Read
* Walk
* Stat
* Clunk (forget about a fid)
* Walk
* (Repeats the Stat, Clunk, Walk) until its done.

The `pcap` file is [provided][p9-pcap], if you want to look at it in Wireshark yourself.
Configure it to Decode TCP port `8999` as 9P.

## Future
* Handle larger files with the `read()` call as it currently can only deal with
  a single block.
* Be able to create an empty image - i.e. equivalent of `mkfs.minix` but using
  a file rather than block device.
* Be able to create directories and files.

[0]: {% post_url 2025-12-23-minix-filesystem %}
[1]: https://filesystem-spec.readthedocs.io/en/latest/
[2]: https://man.cat-v.org/plan_9/5/intro
[3]: https://docs.python.org/3.13/library/os.html#os.walk
[4]: https://docs.python.org/3.13/library/os.html#os.scandir
[5]: https://www.kernel.org/doc/html/latest/filesystems/9p.html
[6]: https://github.com/pbchekin/p9fs-py
[7]: https://github.com/svinota/pyroute2
[p9-pcap]: /assets/minix-p9-demo.pcapng
