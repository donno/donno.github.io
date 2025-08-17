---
layout: post
title:  "Exploring gitremote-helpers protocol"
date:   2025-08-17 20:00:00 +0930
---

When working with Git remotes, the three most common protocols to use are
HTTPS, git and SSH. Git is not limited to just those as there is the ability
to extend this to some other protocol. To do this you write a program
called `git-remote-<transport>`, where `git clone <transport>://...` will cause
that program to be used to fetch references and objects from the that
location.

The [offical documentation][0] describes the overall process as well as the
details of the protocol. As mentioned above, `git` will spawn a helper program
to deal with transport that it doesn't handle. It then sends commands via
standard input and expects results from standard output.

The first command `git` will issue is the `capabilities` command which helps
git know what other command it will expect. This list is not complete as some
capabilities implies that one or more commands will be supported.

## Basics
Essentially, the minimum set of commands to support are these ones:
* `list` - list references
* `fetch` - fetch a reference
* `push` - push a reference.
* `option` - set an option.

I started out looking at `connect` and `stateless-connect`.
The `connect` command requires a bi-directional connection and is built on top
of `receive-pack` and `upload-pack`. The stateless version is marked as
experimental and for internal use only, and uses git's wire-protocol version 2.
The former one is likely better option if you can control the server side.

## Cloning
Running the command:
`git clone base://foo/bar`

This will invoke:
`git-remote-base <remote-name> <remote-url>`
Which for the above command will be:
`git-remote-base origin base://foo/bar`
As `origin` is the default name for the remote.

Additional the git directory - defined by the environment variable called
`GIT_DIR` will be set to the `<cwd>/bar/.git` if you provided the name of the
destination directory so `git clone base://foo/bar bar_01` then it would be set
to `<cwd>/bar_01/.git`.

The messages for performing a clone are as follows:
1. capabilities
2. option progress true
3. option verbosity 1
4. object-format true
5. list
6. option check-connectivity true
7. option cloning true
8. fetch <commit> <ref>

## Pushing
Running the commands
```sh
git remote add base-test base://fs/example
git push base-test main
```

The remote name will be `base-test` in this case and remote URL is
`base://fs/example`. The GIT_DIR environment variable will be the directory
to the git directory for that repository.

The messages for performing a push are as follows:
1. capabilities
2. option progress true
3. option verbosity 1
4. option object-format true
5. push refs/heads/main:refs/heads/main


## Prior Art
Amazon have [shared][1] a git remote helper that uses S3 called
`git-remote-s3`.

Their implementation does not store objects as loose or packs but instead as
bundles, when you perform a push they save a bundle of the reference you are
pushing and upload the bundle.

## Code
The code that corresponds to the generic implementation of the remote helper
protocol itself is as follows in my implementation.
```rust
pub enum SetOptionResult {
    Ok,
    Unsupported,
    Error { message: String },
}

pub struct Reference {
    pub hash: String,
    pub name: String,
}

pub trait Command {
    // Sets the transport helper option <name> to <value>.
    fn set_option(&mut self, name: &str, value: &str) -> SetOptionResult;

    // Lists the references.
    //
    // The output is one per line, in the format "<value> <name> [<attr> ...]".
    fn list_references(&self) -> Vec<Reference>;

    // Fetches the given object, writing the necessary objects to the database.
    // TODO: expand this to include the path to the database.
    fn fetch_object(&self, hash: &str, name: &str);

    // Pushes the given local <src> commit or branch to the remote branch described by <dst>.
    //
    // Discover remote refs and push local commits and the history leading up to them to new or
    // existing remote refs.
    fn push(&self, source: &str, destination: &str, force_update: bool);
}
```

The above means the following capabilities are sent:
* option
* fetch
* push
* object-format

```rust
pub fn handle_command(line: &str, handler: &mut impl Command) -> bool {
    // Read the command from the line and call the corresponding function
    // on handler.

    // <implementation omitted>.
}
```
## Brining it together

To check that it all worked, a `FileBackedCommandHandler` was developed which
essentially uses another git repository on disk to act as the remote.

* `set_option()` simply tags teh value and stores in the map for now.
  A better option would likely be to have a `struct` which stores all the known
  options as documented.
* `list_references()`, looks in the `refs` folder of the repository acting as
  the remote for references and reads the hash within. It doesn't handle
  `packed-refs`.


### Fetch

1. Check if the remote repository has the given object as a loose object.
2. If its not there, then it assumed to be in a pack file and all packs are
   copied.
3. Otherwise, copy that loose object to the local repository if its not already
  a loose object there.
4. Collect references of objects from the loose object.
    * Read the hash of the tree and parent of the commit object.
    * Read the hash of blobs and trees from a tree object
5. Go back to 1 and check each of the referenced objects to see if they need
   to be fetched.

Something I didn't handle is batched fetching as multiple `fetch` commands
can be sent one after another and a blank line marks the end of the batch.
My program didn't handle this case.

### Push

Original implementation

1. Look-up object ID (hash) of the reference in the local repository.
2. Add that object ID to the list of objects to push.
3. Check if next object to push is a loose object in the remote.
4. If it is, then skip it
5. If it is a loose object in the local repository then copy it
  to the remote's object database.
6. Collect referenced objects of the loose object and add them to the list of
   objects to push.
7. If it is not a loose object in the local repository then copy
   all the packs to the remote. This is a quick and dirty trick to avoid having
   to deal with finding out which pack file is needed.
8. Go back to 3, and repeat until all objects needed are pushed.

Reworked implementation

1. Find all objects to push
2. For each object to push:
    * If the object to push is loose, copy that loose object from local to
      remote.
    * If the object to push is pack file including the pack file index, copy it
      from the local repository to the remote repository.

Where the first step is pretty much the same as the above however instead of
copying things one at a time it determines everything that needs copying up
front.

The reason the fetch doesn't do the same approach is because to find the
objects needed, it needs to be able to either read the objects or have some
other  way to look-up the list of objects. While that would work for this file
system backend it doesn't work for say S3 or HTTP one as you end up having to
download the objects anyway.

## Experiment

During the process, I did a few other experiments.

`git remote-https origin https://github.com/git/git.git`

Send: `capabilities`
Response:
```
stateless-connect
fetch
get
option
push
check-connectivity
object-format
```

The above won't work on Windows at least not from PowerShell, as it pressing
enter results in a \r\n (or carriage-return line-feed) which gets rejected by
the `git-remote-curl` helper.

The `list` command doesn't work there but using `git-remote-ls-remote` does
provide similar output, so `git ls-remote https://github.com/git/git.git`.

### Dumb Protocol
The [dumb protocol][2] doesn't work on GitHub, which they stopped supporting
back in [2011][3]. This is way for a simple HTTP server to be able to serve a
repository for `git` without needing any dynamic server-side software for
read-only access.

As documented above you can make a HTTP request to query what references there
are, as well as what the HEAD is and fetch objects.

When I came across this I originally thought of using the reference query
as a quick way to bootstrap teh list reference function so I could focus on the
other components. Once I had realised the idea of having a simple file system
backend to test the program it was easier to simply to read it from an existing
.git directory on disk.


#### Examples
In these examples the git repository used as for the examples is:
```
https://git.kernel.org/pub/scm/infra/cgit.git
```

While writing up this post, I was unable to fetch a loose object. When I first
tried this I tried a project from [sourcehut](4) as that forge/platform supports
the protocol.

* Query the references
  `https://git.kernel.org/pub/scm/infra/cgit.git/info/refs`
  For example:
  * ```
d8e5fdaa4f1f0d39ada860bba2f88bdae14c72ed        refs/tags/v1.0
a6572ce1762e0d571c3e96b5f4eff7c81015a1f2        refs/tags/v1.0^{}
    ```
* Query HEAD
  `https://git.kernel.org/pub/scm/infra/cgit.git/HEAD`
* Query object
  `https://git.kernel.org/pub/scm/infra/cgit.git/objects/a6/572ce1762e0d571c3e96b5f4eff7c81015a1f2`
  * This corresponds to the commit for the tag v1.0.
  * This object does not exist (404).
  * One that does exist was the commit of master, which is 00ecfaadea2c40cc62b7a43e246384329e6ddb98.
    `https://git.kernel.org/pub/scm/infra/cgit.git/objects/00/ecfaadea2c40cc62b7a43e246384329e6ddb98`
* Query if there is another place for the objects to be.
  `https://git.kernel.org/pub/scm/infra/cgit.git/objects/info/http-alternates`.
  * The use case for this is if you had have forks, then you can refer to the
    fork for fetching common objects.
* Query what packs are available.
  `https://git.kernel.org/pub/scm/infra/cgit.git/objects/info/packs`
  * If there are you can then fetch the index to check if the object is in the
    pack (replace the .pack suffix with idx).
    `https://git.kernel.org/pub/scm/infra/cgit.git/objects/pack/<pack name>.idx`
  * If the pack contains the object you were looking for fetch the pack.
    `https://git.kernel.org/pub/scm/infra/cgit.git/objects/pack/<pack name>.pack`

## End-result

I had the building blocks for writing a git remote helper. I also added
ability to parse a `tree` and `commit` object from git to find the objects
they reference.

My example program could be used by defining an environment variable called
`GIT_SOURCE_DIRECTORY` to the path to directories containing a `.git`
directory. Ideally, these would be bare repositories.

The following were then usable.
* Cloning
  `git clone base://git/<name>`
  Where `<name>` is the name of a directory in `$GIT_SOURCE_DIRECTORY`.
* Pushing to a new repository
  1. Create an empty 'remote' repository to test it with.
    ```
    cd "$GIT_SOURCE_DIRECTORY"
    git init --bare base-push/.git
    ```
   The reason it iss `base-push/.git` rather than `base-push.git` is simply because
   the implementation is simply expecting `.git` to be a subdirectory.
  2. Add the remote and push.
    ```
    git remote add fresh base://git/base-push
    git push fresh main
    ```

As above I named the git remote `git-remote-base` as the idea was it provide
a base implementation of the remote helper, in practice that should have been
the name of the library and the binary should be something along the lines of
 `git-remote-fs`.

## Future

* Exploring the smart protocol.
* Explore reading the index file for pack file.

[0]: https://git-scm.com/docs/gitremote-helpers
[1]: https://github.com/awslabs/git-remote-s3
[2]: https://git-scm.com/book/en/v2/Git-Internals-Transfer-Protocols
[3]: https://github.blog/news-insights/git-dumb-http-transport-to-be-turned-off-in-90-days/
[4]: https://sourcehut.org/
