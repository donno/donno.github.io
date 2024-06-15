---
layout: post
title:  "BitTorrent in Containers"
date:   2024-04-20 20:00:00 +0930
---

The premise of this project came about the idea of imitate a network such as
a mini-internet, with DNS, web hosts and some kind of simulated traffic between
them. That project would have been too ambiguous so reducing it down to use
stuff that already exists and can already be automated was needed.
The existing tools that fit the brief was BitTorrent.

BitTorrent handles the sharing of information between peers (computers),
a client can handle automatically participating in the information transfer by
receiving a torrent file and a tracker is simpler to set up then DNS server.
The peers and tracker are each their own container.

Now that I had an idea in behind that seemed easier to accomplish in a weekend
or two, I began setting it up.

## Tracker
First, I set out to find a BitTorrent tracker. I came across many that were
written in PHP and required web server, etc to be set up. The desire was to
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
photos of a mobile phone application testing station.

### Seeder
Now that there was a tracker and some torrents. For the torrent client, the
options quickly dwindled down to [rtorrent][7].

The plan was to provide the torrent files and the source files to a container
with the torrent client to act as a seeder. The act of seeding is where
already downloaded content is uploaded to others to download.

Trying to get rtorrent set-up is where much of the time in the project was
spent.

An issue that was encountered was the container saying "error opening
terminal". This turned out to be because it needs to be run in daemon mode
rather than use the ncurses UI as that doesn't work when there is not an
interactive session. The alternative would have been to run it in tmux.

I started out manually loading the torrent which is where I discovered the
tracker address was wrong as the /announce path fragment was missing from
tracker URL, so it was unable to see the tracker. It enabled to be confirm it
was able to load the files.

In between this another torrent client was tried which had a web-ui built-iin.
This client was Deluge, I think (I wrote this post several weeks after). While
this made it easy to load the torrent file, what it didn't do was recognise I
already had the file and instead it downloaded it on its own. This was possible
because that client had DHT (distributed hash table) protocol enabled and
because I had used a file that other people share it was able to connect to
peers and download the file.

After a great deal of mucking about, it turned out the thing that made it work was
creating a `.rtorrent.rc` which is added to the container instead of trying to
configure it via the command line options. Choosing this approach from the
start would have saved time as it gives you full control over the settings.

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

There wasn't time that night to get started on the client as I needed sleep.
It however felt very close to being within reach of having it all working.

### Client
Now that rtorrent was already set-up, all that was left to be done was set-up
another container which has rtorrent and has the torrents but not the files.

This was straight forward now that rtorrent had been smoothed out and was
working. The main difference between it and the seeders is having a different
download location which meant simply not mounting the sources to the download
location but still mounting the torrents in to the watch directory.

```yaml
  leachers:
    # This only has the torrents and no sources.
    build:
      dockerfile: Dockerfile.client
      target: client
    volumes:
      - torrents:/root/rtorrent/watch/start
    deploy:
      mode: replicated
      replicas: 10

    networks:
      - torrenting
```

The separation of the leachers and seeders onto a torrenting network was done
once I knew that torrenting part had ben setup.

### User Interface
When trying to get a better sense of what the seeder was doing, I tried out
[Flood](9) which is a web interface for rtorrent. This worked well when running
it in the same container as rtorrent, so initially I imply set-up a seperate
target in the Dockerfile which installed flood from npm.

Eventually, I reworked it so there was a single container with flood and
service in the compose file which hosted flood and set it up to expose the RPC
port to that container.

I started off with the idea of having a volume which contained an RPC socket
for each rtorrent client didn't work. The way to make it worked was to use the
TCP, which ideally would be it opt-in / configurable.

For example of the RPC socket file approach:
```
network.scgi.open_local = (cat, /rpc_sockets/rpc_, (system.env, HOSTNAME), .socket)
```
Where /rpc_sockets was a volume attached to the leachers, seeders and flood
service.

### Closing Remarks
It look me a while to write this up so there is likely some things I
overlooked. The idea was to publish this anyway even if it was incomplete.

In the end, the compose file, starts a service to create the torrents and
two seeders which had access to the original files followed by 10 leachers.
Additionally, it can start the seeder-ui service with [flood](9) if the debug
profile is used.

The future idea would be write a custom front-end for monitoring multiple
rtorrent's all at once. Extra nice would be if it was possible to show what
blocks came from which peers.


[0]: https://erdgeist.org/arts/software/opentracker/
[1]: https://pkgs.alpinelinux.org/package/edge/community/x86_64/opentracker
[2]: https://alpinelinux.org/
[3]: https://github.com/gliderlabs/docker-alpine/issues/307#issuecomment-427256504
[4]: https://github.com/pobrn/mktorrent
[5]: https://pkgs.alpinelinux.org/package/edge/community/armhf/mktorrent
[6]: https://www.man7.org/linux/man-pages/man1/find.1.html
[7]: https://rakshasa.github.io/rtorrent/
[8]: https://pkgs.alpinelinux.org/package/edge/community/x86_64/rtorrent
[9]: https://flood.js.org/