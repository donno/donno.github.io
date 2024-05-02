---
layout: post
title:  "BitTorrent in Containers"
date:   2024-04-20 20:00:00 +0930
---

The premise of this project came about the idea of imitate a network such as
a mini-internet, with DNS, web hosts and some kind of simulated traffic between
them. I quickly realised that project was a little to ambiguous so reducing it
down to use stuff that already exists and can already be automated was needed.
The existing tooling that fit the brief was BitTorrent.

BitTorrent handles the sharing of information between peers (computers),
a client can handle automatically participating in the information transfer by
reviving a torrent file and a tracker is a lot simpler to set-up then DNS
server. The peers and tracker are each their own container.

Now that I had an idea in behind that seemed a lot easier to accomplish in a
weekend or two, I began setting it up.

## Tracker
First, I set out to find a BitTorrent tracker. I came across many that were
written in PHP and required web server, etc to be set-up. The desire was to
have something ready-to-go and quite small. The project [opentracker][0] was
discovered and looked like it was just what this system needed. It comes
as a single executable and simply needs a configuration file. To top it off
a [package][1] was already available for [Alpine Linux][2].


Putting this together lead to the following Dockerfile for the tracker.
```Dockerfile
# syntax=docker/dockerfile:1
FROM alpine:3.19.1
RUN apk add --no-cache opentrackeralpine/v3.19/community

EXPOSE 6969
ENTRYPOINT [ "/usr/bin/opentracker" ]
```

In the file that I use, the command to install apk is slightly more involved
as I run into what seems to be issue [alpine-linux#307][3] where the default
mirror times out.

```Dockerfile
RUN rm -f /etc/apk/repositories && apk add --no-cache opentracker --repository http://mirror.aarnet.edu.au/pub/alpine/v3.19/main --repository http://mirror.aarnet.edu.au/pub/alpine/v3.19/community
```

## Torrent Creation
Now that the tracker is available, the next step is to create a torrent that
makes use of that tracker. This uses [mktorrent][4] which is a single binary
and has a [package][5] for Alpine Linux.

The intent was to produce some torrents and move on. This did not go to plan as
instead a shell script was developed for creating a torrent for each file in
a directory and a single torrent for each directory. The latter case meant
there would torrents would multiple files.

This turned out to be an opportunity to play with [find][6] and meant I didn't
quite get to the torrent client part that evening.


The test data was `alpine-standard-3.19.1-x86_64.iso` and a folder with three
photos of a phone application testing station.

### Seeder
Now that there was a tracker and some torrents.  For the torrent client, the
options quickly dwindled down to [rtorrent][7].

The plan was to provide the torrent files and the source files to a container
with the torrent client to act as a seeder. The act of seeding is where
already downloaded content is uploaded to others to download.

Trying to get rtorrent set-up is where a lot of the time in the project was
spent.

I started out manually loading the torrent which is where I discovered the
tracker address was wrong as the /announce path fragment was missing from
tracker URL, so it was unable to see the tracker. It enabled to be confirm it
was able to load the files.

In amongst this tried another torrent client which had a web-ui built in, which
was Deluge, I think (I wrote this post several weeks after). While this made it
easy to load the torrent file, what it didn't do was recognise I already had 
the file and instead it downloaded it on its own. This was possible because 
the client had DHT (distributed hash table) protocol enabled and because I had
used a file that other people share it was able to connect to peers and
download the file.

After a lot of mucking about, it turned out the thing that made it work was
creating a `.rtorrent.rc` which is added to the container instead of trying to
configure it via the command line options.

The magic part was setting up the watch directories so it looks for torrents
in a certain directory and automatically starts them. Fortunately, it
automatically knows to look to see if they already in the download folder and
if they are there it will start seeding that (or if it was partial resume it).

```Dockerfile
# syntax=docker/dockerfile:1
FROM alpine:3.19.1 as client
# The default mirror/CDN Fastly has issues with MTU size.
# https://github.com/gliderlabs/docker-alpine/issues/307#issuecomment-427256504
# To work around this, pull from a mirror in my home country.
RUN apk add --no-cache rtorrent
ADD .rtorrent.rc /root
EXPOSE 5000
CMD [ "/usr/bin/rtorrent", \
      "-o", "system.daemon.set=true", \
      "-o", "network.scgi.open_port=0.0.0.0:5000" ]
```

Most of the RC file was setting-up the different file paths and creating the
folders. The relevant part was:
```
## Watch directories (add more as you like, but use unique schedule names)
## Add torrent
schedule2 = watch_load, 11, 10, ((load.verbose, (cat, (cfg.watch), "load/*.torrent")))
## Add & download straight away
schedule2 = watch_start, 10, 10, ((load.start_verbose, (cat, (cfg.watch), "start/*.torrent")))
```

I didn't have time that night to get started on the client as I needed sleep.
It however felt very close to being within reach of having it all working.

### Client
Now that rtorrent was basically set-up, all that was to be done was set-up
another container which has rtorrent and has the torrents but not the files.


### DRAFT

Things to cover/mention:

- The challenge was getting rtorrent to load the torrents for the seeder. In
  the end the best thing was to to create the .rtorrent.rc to give full control
  over the settings.
- Missing the /announce on the tracker URL.
- The issue with the container saying "error opening terminal" was because it
  needs to be run in daemon mode rather than use the ncurses UI as that doesn't
  work when there is not an interactive session. The alternative would have
  been to run it in tmux.
- - Tried out Flood as a web UI. Worked well but for now running it in same
  container as rtorrent.
  - This has some potential for being quite neat for monitoring the set-up.
- The idea of having a volume which contained an RPC socket for each rtorrent
  client didn't work. I think the best thing to do there would be to use the 
  TCP option. Ideally make it opt-in.
  ```
  network.scgi.open_local = (cat, /rpc_sockets/rpc_, (system.env, HOSTNAME), .socket)
  ```
- 

[0]: https://erdgeist.org/arts/software/opentracker/
[1]: https://pkgs.alpinelinux.org/package/edge/community/x86_64/opentracker
[2]: https://alpinelinux.org/
[3]: https://github.com/gliderlabs/docker-alpine/issues/307#issuecomment-427256504
[4]: https://github.com/pobrn/mktorrent
[5]: https://pkgs.alpinelinux.org/package/edge/community/armhf/mktorrent
[6]: https://www.man7.org/linux/man-pages/man1/find.1.html
[7]: https://rakshasa.github.io/rtorrent/
[8]: https://pkgs.alpinelinux.org/package/edge/community/x86_64/rtorrent
