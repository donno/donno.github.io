---
layout: post
title:  "Minix filesystem"
date:   2025-12-23 22:10:00 +1030
---

After watching the talk ["50 years in filesystems"][0] by Kristian Köhntopp
given at [FrOSCon][1] 2025, I was motivated to take a look at how format of
file systems. Rather than focus on Ext3 or a newer system, I stuck with the
file system used by Minix which had been simplified for teaching anyway.

This began with reading the file system section the book "Operating Systems:
Design and Implementation" by Andrew S. Tanebaum, which had been sitting in the
bookshelf for quite a while since hte last time I picked it up. The edition
was copyrighted in 1987 and listed no other dates in its cover.

This post is more about the bytes and representation of the data on disk
rather than about the higher-level structures.

## Building file system image

To look at the bytes of the file system, it helps if there is an existing
system to look at. In this case, we can build one.

The following was done on ALpine Linux and creates a 25MB image that is
formatted with the Minix file system.

```sh
apk add util-linux-misc
dd if=/dev/zero of=minixfs.raw bs=256K count=100
losetup /dev/loop0 minixfs.raw
mkfs.minix -1 /dev/loop0
losetup -d /dev/loop0
```

A breakdown of what the above is doing is
1. Installing the package which contains the `mkfs.minix` program
2. Creating a file that is 25MiB filled with zeros.
  It is 25MiB because block size (`bs`) is 256K meaning 257 kibibytes with
  a count of 100.
3. Associate the file with a loop device.
4. Format the specified device (the loop device above) as Minix. The -1 means
   we want version 1 (there is three versions it can be). If
   `mkfs.vfat` was used then it would be a FAT32 file system instead.
5. Disassociated the loop device.

You can also confirm the size of the 'device' while its still associated to the
loop like so:
```
$ fdisk -l /dev/loop0
Disk /dev/loop0: 25 MiB, 26214400 bytes, 51200 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```

The next stage, would be to mount the system and add directories and files
to it. I haven't done at at this stage, which is a problem the file system
doesn't contain any directories other than root.

This was problematic:
```
[ ! -d /media/minix ] && mkdir /media/minix
mount /dev/loop0  /media/minix/
mount: /media/minix: unknown filesystem type 'minix'.
       dmesg(1) may have more information after failed mount system call.
```

Likewise: `mount /dev/loop0 /media/minix/`

I suspect the issue is I'm using Windows Subsystem for Linux and its kernel
doesn't have the module for reading `minix`. The solution in the end was
to boot up a VM using the Ubuntu Server in live mode. The main difference is
needed to use `losetup --find minixfs.raw` as the live version mounts several
squashfs images and snaps.

### Setup Contents

The contents of the image was set-up as follows:
```sh
cd /media/minix
mkdir system users media
mkdir users/dick users/erik users/jim users/ast media/audio media/videos
mkdir system/bin system/lib system/etc system/var
touch system/bin/sh system/bin/ls system/bin/rm system/bin/ping system/bin/tar
touch media/videos/big-buck-bunny.mkv media/audio/fur-elise.mid
echo "Hello" > users/ast/welcome
echo -e "OS: Design and Implementation\nTo Kill a Mocking Bird" >> users/ast/books
```

## Structures

The start of the disk:

* Boot block - hardware reads the block to memory then jumps to it, or at least
  that is the idea.
* Super block - Describes the layout of the system
* I-Node bitmap
* Zone bitmap
* I-Nodes
* Data

An I-node is an "index node" as it is a node in the index of the file system,
however it more widely known as simply an `inode`.

### Super Block
Stores important information so the system can locate and understand the file\
system.

|Name        |Type                   | Description                            |
|------------|-----------------------|----------------------------------------|
|I-Node Count|16-bit unsigned integer|The total number of i-nodes.            |
|Zone Count  |16-bit unsigned integer|The total number of zones.              |
|I-Node bitmap block Count|16-bit unsigned integer|The total number of blocks that form the I-Node bitmap. |
|Zone bitmap block Count|16-bit unsigned integer|The total number of blocks that form the zone bitmap. |
|First Zone  | 16-bit unsigned integer|-|
|Log Zone Size | 16-bit unsigned integer|The log2 of block size.|
|File Size Max|16-bit unsigned integer|The maximum size (in bytes) of a single file.|
|Magic|16-bit unsigned integer|The magic number that represents this file system.|

### Magic Numbers
* Version 1: `0x137f`
* Version 2: `0x2468`
* Version 3: `0x4d5a`

The book only covers version 1 and I was only focusing on that.

### Directory

A directory contains the entries of the directory, which are made up of
the I-node number and a name.

## Functions

These are the functions that the Operating System: Design and Implementation
book encourages and are still used in the Minix 3.

I won't be going into a deep dive of these functions today, however I am keen
to stick with modeling the access of the file system using these functions.

### Block Management

* `get_block`- Fetch block for read/write
* `put_block`- Return a block that was fetched by `get_block()_`.
* `alloc_block` -  Allocate a new zone (make a file longer).
* `free_block` -Release a zone (a file is removed).
* `rw_block` - Transfer block between disk and cache.
* `invalidate` - Purge all cached blocks.

### I-node Management

* `get_inode` - Fetch an i-node into memory.
* `put_inode` - Indicate that an inode is no longer needed in memory.
* `alloc_inode` - Allocate a new inode (for new file).
* `wipe_inode` - Clear some some fields of a newly allocated inode.
* `free_inode`-  Releases an i-node (file removed)
* `rw_inode` - Transfer an i-node between memory and
* `dup_inode` - Indicate that someone else is using an i-node table entry. This
  increases the usage count.

## Example
Let us start from the file system view itself, which is made up of files
and special files called directories.

The end goal is you want to load contents of a file called `/usr/ast/welcome`
which contains the bytes "Hello".

### User abstraction
For the end user, there is this:
```
$ ls /
.
..
bin
dev
lib
usr

$ ls /usr
.
..
dick
erik
jim
ast

$ ls /usr/ast
.
..
welcome
books
```

### File system abstraction

For this example, there is only three directories that are important and one
file
* Root (/)
* /usr
* /usr/ast
* /usr/ast/welcome

#### Root Directory
The numbers are the i-node numbers for the i-node that corresponds with the
given file (or directory). The numbers may be different on your system, these
were from the book.
* 1 .
* 1 ..
* 4 bin
* 7 dev
* 14 lib
* 6 usr

### I-Node 6 (/usr)
The relevant part here is the zone/block number which states which block the
directory entry is in.

### /usr Directory
This is I-node 6, which is why the self entry points to 6.

* 6 .
* 1 ..
* 19 dick
* 30 erik
* 51 djim
* 26 ast

### I-Node 26 (/usr/ast)
The relevant part here is the zone/block number which states which block the
directory entry is in.

### /usr/ast Directory

* 26 .
* 6 ..
* 64 welcome
* 92 books

### I-Node 64 (/usr/ast)
The relevant part here is the zone/block numbers.

## Code

The process of implementing this was:

1. Define several of the key data structures
2. Read the super block from the file
3. Read the root i-node.
    1. Transfer a block from disk into memory (i.e. read a block from the image).
    2. Decode the i-node.
4. Read the directory entries provided by the root i-node.
    1. For each zone number
    2. Load block
    3. Decode the directory entries.

### Read Directory

```python
@dataclass
class DirectoryEntry:
    """Associates a filename with an inode."""

    inode_number: int
    """The corresponding i-node number for the child (file)."""

    filename: str
    """The name of the file (including if its a directory)."""


@dataclass
class Directory:
    """A node within the file system index which represents a directory.

    This includes its children.
    """

    node: IndexNode
    entries: list[DirectoryEntry]

class FileSystem:
    ...
    def read_directory(self, inode: IndexNode) -> Directory:
        """Read the directory referred to by the given inode."""
        if (inode.mode & INODE_TYPE_MASK) != ModeFlags.DIRECTORY
            raise ValueError("The inode should be a directory but wasn't.")

        if inode.indirect != 0:
            raise ValueError("Directories with indirect i-nodes is NYI.")
        if inode.double_index != 0:
            raise ValueError("Directories with double-index i-nodes are NYI.")

        entries = []
        for zone in inode.zone_numbers:
            block = self.get_block(zone)
            for child_inode, name in struct.iter_unpack("<H14s", block):
                if child_inode != 0:
                    terminator = name.index(b"\0")
                    name = name[:terminator]
                    entries.append(DirectoryEntry(child_inode, name))

        return Directory(inode, entries)
```

This probably should be reading the file size from the I-node to know when to
stop.

### Open Image
The function for opening the image and reading it currently looks like this,
where its st ill incomplete as ideally it would return something a
`OpenedImage` object or something as well as functions for working with the
file system but for now the focus is just on reading it.

```python
def open_image(path: pathlib.Path):
    """Open an image that was formatted as the Minix file system.

    For a fresh, empty Minix file system the following can be run in Alpine
    Linux to create a disk image that is 25MB and formatted as a Minix
    filesystem.

        apk add util-linux-misc
        dd if=/dev/zero of=minixfs.raw bs=256K count=100
        losetup /dev/loop0 minixfs.raw
        mkfs.minix /dev/loop0

    The information output by mkfs.minix is stored in the super block.
    """
    with path.open("rb") as reader:
        system = LoadedSystem.from_reader(reader)
        print(system.super_block)

        root_directory = system.read_directory(system.root_inode)
        print("Root I-Node", root_directory.node)
        print("Root director entries:\n  " +
              "\n  ".join(map(str, root_directory.entries)))
        print(f"Cleanly unmounted: {system.super_block.cleanly_unmounted}")
```

Using this on the image created above gives:
```
SuperBlock(inode_count=8544, zone_count=25600, inode_map_block_count=2, zone_map_block_count=4, first_zone=275, log_zone_size=0, file_size_max=268966912, magic=5007, state=1)
Root I-Node IndexNode(mode=16877, uid=0, file_size=160, time_of_last_modification=0, links=87, gid=108, zone_numbers=(1280, 275, 0, 0, 0, 0, 0), indirect=0, double_index=0)
Root director entries:
  DirectoryEntry(inode_number=1, filename=b'.')
  DirectoryEntry(inode_number=1, filename=b'..')
  DirectoryEntry(inode_number=2, filename=b'system')
  DirectoryEntry(inode_number=3, filename=b'users')
  DirectoryEntry(inode_number=4, filename=b'media')
```

### Reading File
At this point, I really wanted to implement a function equivalent to
[`os.walk`][5], which generate the file names in a directory tree by walking
the tree either top-down or bottom-up but instead I wanted to simply focus on
ensuring a file can be read.

For this set-out on writing a function to make this easier:
`def _path_to_inode(path: pathlib.PosixPath) -> IndexNode:`

As the signature suggests, the idea would be given the path look-up the inode
for the final path, it could query the number instead but since it is Python
might as well refer to the `IndexNode` object instead.

```python
class FileSystem:
    ...

    def _path_to_inode(self, path: pathlib.PurePosixPath) -> IndexNode:
        if not path.is_absolute():
            raise ValueError("Relative paths is NYI or supported.")

        assert path.parts[0] == '/', "Only absolute paths are supported"
        parts = path.parts[1:]
        current_node = self.root_inode
        path_so_far = pathlib.PurePosixPath("/")
        for part in parts:
            children = self.read_directory(current_node).entries
            path_so_far = path_so_far / part
            try:
                part_inode_number = next(
                    child.inode_number
                    for child in children
                    if child.filename == part.encode('utf-8')
                )
            except StopIteration:
                message = f"Couldn't find {path_so_far}"
                raise FileNotFoundError(message) from None

            # Look-up the inode.
            current_node = self.get_inode(part_inode_number)

        return current_node

    def read(self, path: pathlib.PurePosixPath | str) -> bytes:
        """Read the contents of a file at the given path."""
        # TODO: Rework this so its more of an "open" and is its own context
        # manager and provides the file-like interface over the underlying file.
        inode = self._path_to_inode(pathlib.PurePosixPath(path))

        if (inode.mode & INODE_TYPE_MASK) == ModeFlags.DIRECTORY:
            raise ValueError("Not expecting a directory.")

        if (inode.mode & INODE_TYPE_MASK) != ModeFlags.REGULAR:
            raise ValueError("Not a regular file - only regular files are supported for now.")

        if inode.indirect != 0:
            raise ValueError("File with indirect i-nodes is NYI.")
        if inode.double_index != 0:
            raise ValueError("Files with double-index i-nodes are NYI.")

        def position_to_block_number(position: int) -> int:
            """Given the position within the file return the block number in
            which that position is found.

            This is the block not zone number.
            """
            block_index = position // BLOCK_SIZE
            zone_index = block_index >> self.super_block.log_zone_size
            block_offset = block_index - (zone_index << self.super_block.log_zone_size)
            assert zone_index < len(inode.zone_numbers)
            zone_number = inode.zone_numbers[zone_index]
            if zone_number == 0:
                raise ValueError("No zone or block number")
            return (zone_number << self.super_block.log_zone_size) + block_offset

        # As above, this would ideally be generalised to be more like
        # read(..., size) based on current position, so lets pretend that is
        # what is implemented here.

        def _read(position: int, size: int) ->  bytes:
            """Read the given number of bytes (size) starting at position."""
            block_number = position_to_block_number(position)
            block = self.get_block(block_number)
            if len(block) < size:
                raise ValueError("Only reading the first file is supported")
            return block[position:position + size]

        return _read(position=0, size=inode.file_size)

```

When trying to read the file, I had realised there was problem reading
the I-node structure, as to get the file to read correctly I needed to read
from the second zone number. The problem was I hadn't accounted for the
file size and time being 32-bit (4 bytes each), in the book its spaced higher
but I forgot to confirm that was on purpose.

The result was with:
```python
    with path.open("rb") as reader:
        system = LoadedSystem.from_reader(reader)
        print("Contents of /users/ast/welcome", system.read("/users/ast/welcome"))
        print("Contents of /users/ast/books", system.read("/users/ast/books"))
```

The output was:
```
Contents of /users/ast/welcome b'Hello\n'
Contents of /users/ast/books b'OS: Design and Implementation\nTo Kill a Mocking Bird\n'
```

## Missing
I haven't gotten around to sorting out:
* Decoding the permissions
* What the bitmaps are for - I suspect its more for writing as it likely part
  of the "free/used list".
* Difference between a zone and block - due to the struggle with the reading I
  ended up going deeper on that in the read function.

## Future
Still got a bit more to go and various ideas that I would like to explore.

A few things would be:
- Add a walk function to walk over the entire file system. A similar idea would
  be similar to `find` command but with verbose output to show permissions etc.
- Writing the file system
- Write a tool that converted a `tar` to a Minix formatted disk image.
- Separate block handling / device handling out - i.e. try a more layered
  approach
- Implement [`fsspec`][6] over which provides a more common interface over it
  the higher level file system interface. Think of it as `pathlib` but for the
  IO.

A look into the FAT (File Allocation Table) file system is something I'm keen
on as well.

Another idea is write a specification of Minix file system using
[Kaitai Struct][3], as they already have `vfat` and `ext2` in their [format
gallery][4].

[0]: https://media.ccc.de/v/froscon2025-3238-50_years_in_filesystems/
[1]: https://froscon.org/en/
[3]: https://kaitai.io/
[4]: https://formats.kaitai.io/
[5]: https://docs.python.org/3.9/library/os.html#os.walk
[6]: https://filesystem-spec.readthedocs.io/en/latest/
