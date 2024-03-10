---
layout: post
title:  "Streaming containers"
date:   2024-03-03 20:00:00 +0930
---

A friend of mine had an idea for a "Twitch Plays" style of stream. This is the
journey of setting-up a machine to run it. The plan was to start with an
existing machine running Microsoft Windows and use OBS Studio. Not all plans
survive first contact, as my Raspberry Pi idea didn't work either.

## Content of Stream
As mentioned, the idea was a "Twitch Plays" style of stream. The three parts
of this are rendering/drawing the content, handling viewer interaction and
then streaming it to Twitch. The streaming part had been on my mind for a
while. I've had my own idea for a project that would make a nice live-streaming
idea. That idea has many parts that the streaming is only a small component of
it. This friend's idea had given me a smaller project to work that would help
me solve problems of my own project. The fact it took viewer interaction
required a bit more setup to test and wasn't worth simulating.

To explore the streaming component, I need something that already existed.
Without doing this the incomplete projects would get in the way. The brainwave
I had was there are chess games played online which others can spectate. The
spectating is even done via a website. As a proof-of-concept, the goal would be
to stream the web page of these games. This eliminates the content generation
aspect while focusing on the streaming side.

Enter Lichess TV at https://lichess.org/tv which handled the content
generation (video and could do sound too). It already has the viewer, so there
no need to build a separate program for rendering the game state. Now I had
what I wanted to stream the next part was hooking it up and a machine that
could ideally run it 24/7.

## Rapsberry Pi
I first tried running it on a Raspberry Pi 3 Model B Rev 1.2 that is running
Raspbian GNU/Linux 12 (bookworm). This could not keep up, and would start
dropping frames and lagging out the machine to the point where it stopped
allowing input.

The prototype I built for the friend's idea above was a simple web page, so
what needed to be running was a browser and ffmpeg. I tried both
Chromimum and Firefox but neither performed well enough.

At this point, I realised I need another machine. I started by looking up
purchasing a mini-pc form factor which I been looking at getting for a while
for another project. In the end, I discovered my fiancé had an old unused
machine sitting around in the house which could be used.

## Old Windows PC
The Windows machine was running Microsoft Windows 8.1 and took a day to log in
to as the expected set of passwords it could have used weren't working.
Despite, Windows trying to be helpful and say we could reset the password
online, it turned out the password was not the same as the Microsoft account.Several of the old tricks to enable local administrator didn't work. In the
end happened to remember it.

I loaded on OBS (Open Broadcaster Software) Studio and then discovered that
was a no-go. The problem was it uses `IsWow64Process2()` from
`kernel32.dll`. That particular function was new in Windows 10, version 1709.
From my quick look over into it why it was using it it seemed the function is
used to determine if the process is being run on a 64-bit ARM machine.

The four options that I considered were:
- Patch it so it doesn't make that function call. Given that it is to fix an
  unsupported operating system there would be [no interest][0] in a pull
  request for it. The risk is there might be more things lurking.
- Downgrade to a version of OBS Studio which ran on Windows 8.1. At the time
  I didn't go hunting but for those wondering, it seems the [answer][1] to the
  question is that is version 27.
- Try the ffmpeg method instead.
- Don't use Windows 8.1.

I chose the last option, as we had missed the window for the free Windows 10
upgrade. The timing of this happened to be bad because I had an SSD lying
around unused for years. The drive had Windows 10 installed on it and I
formatted it four weeks prior.

I tried to boot an old Ubuntu 10.04 CD that I had without much success and then
I briefly considered setting up network (PXE) boot. After that I tried booting
Alpine Linux from a USB. This is where I realised the reason Ubuntu and PXE
had issues was that SecureBoot was enabled on the machine.

After disabling SecureBoot, I was able to get into AlpineLinux.

## AlpineLinux
I had used [Rufus][2] to prepare the bootable USB from the ISO which does what
seems to be called a diskless or data disk mode install. This means unlike
a traditional install where the system is installed directly onto the USB
device with the root filesystem, the OS runs from RAM and it uses folders on
the USB to storage persistent data.

In preparing this article, I've since discovered that this means that
`setup-lbu` abd `lbu commit` is needed to to configure location for the
diskless system and to save the local configuration state respectively. This
is something I didn't do and as a result when I rebooted needed to redo
everything.

I started off down a bad path by trying to be brave and use Wayland and was
unable to get a graphical system set-up, I restarted and went with X. Alpine
comes with a script to get this going called `setup-xorg-base`. I initially
tried installing parts on my own and while I was able to see a xterm window,
I was unable to type into it or move the mouse as I was missing the right
libinput package, which the setup script handles.

From there I installed the additional packages, `firefox` and `ffmpeg`.
I had originally installed `python3` and `py3-pip` so I could use Selenium to
control the browser (more on that in a bit). I also installed `fluxbox` so I
had basic window management.

At the time, I didn't even consider trying OBS Studio, in fact it hadn't
crossed my mind that it might be available for Alpine.

Anyway, at this point I had the pieces and I needed and was readying to stream,
and stream I did. I won't go into the exact commands here and will save that
for the next section.

Since, my install wasn't persist and I was expecting to have to re-install
everything next time. This wasn't going to be too bad  as I most parts of it
cripted at that point. I thought maybe it would be neat to try building a
container image to stream from.

## Containerisation

The ingredients in this cake or at least the packages that are being loading
into the container are:

- Mozilla Firefox - This is essentially the video source for our stream.
- ffmpeg - This is video capture, transcoding and transmitting of the source to
  Twitch.
- Xvfb - A X virtual framebuffer which is a display server implementing the X11
  display server protocol. This is what Firefox is drawn too and what ffmpeg
  will capture from.
- xdotool - Resize Firefox to be full screen, it seems while kiosk mode is meant
  to make it full screen, the lack of a window manager seems to be stopping it.

### Dockerfile
```Dockerfile
# Building: podman build -t lichess_streamer:latest .
# Running: podman run --env-file twitch_key.sh lichess_streamer:latest
FROM alpine:3.19.1
# Use a mirror close by, the default CDN keeps timing out for me.
ADD repositories /etc/apk/repositories
RUN apk add --no-cache firefox ffmpeg xvfb xdotool
ADD stream.sh run.sh /root
WORKDIR /root
ENTRYPOINT [ "sh", "run.sh" ]
```

For completeness in-case you wondering the contents of `repositories` was
```
https://mirror.aarnet.edu.au/pub/alpine/v3.19/main
https://mirror.aarnet.edu.au/pub/alpine/v3.19/community`
```

### Stream Script
This is the most generic part of the system and could be reused for other
endeavours.

```sh
#!/bin/sh
# Load the stream key from a file if present.
#
# If running as a container instead use --env-file and use your twitch_key.sh
# as the env file instead.
[ -e twitch_key.sh ] && . twitch_key.sh

ENDPOINT=rtmp://syd02.contribute.live-video.net/app/$STREAM_KEY
INPUT="-f x11grab -framerate 25 -video_size hd1080 -i :1.0"
OUTPUT="-vcodec libx264 -b:v 5M -acodec aac -b:a 256k -f flv $ENDPOINT"
ffmpeg $INPUT $OUTPUT
```

### Set-up
```sh
#!/bin/sh
# Maybe use supervisord instead to start the two background tasks.
Xvfb :1 -screen 0 1920x1080x24 &
firefox --kiosk https://lichess.org/tv --display=:1 &

# Resize firefox to take the full view.
DISPLAY=:1 xdotool search --name ".*Mozilla Firefox" windowsize 1920 1080

sh stream.sh
```

### Notes

There were a few things I still need to look at.

* Make the 'browser's address configurable to make it a more general.
* Possibly in my example I could avoid adding the repositories to the container
  and instead use the `--repository` command line argument of `apk add`.
* Allow the ENDPOINT to be configured via envfile as well.
  See [Recommended Ingest Endpoints For You][3] page in the Twitch help for
  list of endpoints.
* The libx264 usage seems to prevent Mozilla Firefox from being able to view
  the stream, I have to use Microsoft Edge to view it instead.
* If the Xvfb set-up could be baked into the image so it always loads on
  start-up that would be nice.

## Turn-based strategy to real-time strategy
Now that I had built the container, I realised I could remix it and try
using it for another game. Chess is a turn-based strategy so I set my sights on
a real-time strategy game. The next experiment was with [OpenRA][4], an
open source project that recreates the classic game of Red Alert by Westwood
(since bought out by Electronic Arts).

The challenge here is when OpenRA first starts it prompts the user to install
content from the original game. Clicking Quick Install then downloads the
original games that were made freely available on the Internet and extracts the
files needed. There does not seem to be a command line flag to say do this
without user input nor is there a separate set-up tool. To solve this, I simply
add the content folder (from a pre-installed game) to the container.

```Dockerfile
# Requires the content folder with the .mix files in v2/.
ADD v2/ /root/.config/openra/Content/ra/v2/
```

Now that got is into the menu the next problem is the menu. Since this was more
of a proof-of-concept I opted to installed x11nvc in the container image such
that I could remote in. From here I manually set-up a bot-match.

I will point out that I switched from Alpine to ArchLinux as was listed on the
official website as having packages available. When writing this, I checked and
it turns out there is a package for Alpine.

### Problems

The first problem is the automation. It looks like this may be possible to
solve by setting-up dedicated server to do bot match and then telling the
client instance to connect it it.

There is Launch.Connect (IP:PORT) and Launch.URI (opeanrea://IP:PORT)
command line arguments to control this. An alternative would be to use
Launch.REplay to play a replay file instead, which would get a proof-of-concept
running.

The dedicated server approach has the benefit that it would be a nice use of
a compose file. The basic idea would be set-up one container/service to run the
dedicated server and another runs the client which then streams.

The remaining issue which outside the replay idea is a major limitation of
an automated approach. That is the lack of an **automated** spectator mode,
which would be like a bot that control the view port and moves to where the
action is happening. This would include the fights and the building progression
as well as resource acquisition. This shares some similarity to the idea I
alluded to above where part of the problem is switching between different 
state or views.

Without the automated spectator you would need a human to move around, which
may not rule this overall idea or container out. A possible use-case for this
container would be someone that would like to develop their craft as a
commentator (sports caster). Of course, the technical challenges here is
rethinking how the feed is mixed so you can add the commentators audio into the
feed and/or have their floating head (overlay). One option would be use OBS and
use the VNC window as the source and second option would be use set-up NGINX's
RTMP Media Streaming Module so the container sends the video thread to NGINX
rather than Twitch and then you pull that feed into OBS. A third option would
send the audio from the microphone into the container to be mixed there and
sent out.

### Dockerfile
```Dockerfile
# Build image with: podman build -t openra-play:archlinux .
# Run image with: podman run --rm -it openra-play:archlinux
FROM archlinux:base
# Or: podman pull quay.io/archlinux/archlinux:latest

RUN pacman --noconfirm -Syu
RUN pacman --noconfirm -S ffmpeg openra xorg-server-xvfb

# Requyires the content folder with the .mix files in v2/.
ADD v2/ /root/.config/openra/Content/ra/v2/

ADD stream.sh run.sh /root/

# For testing, install x11vnc and expose the port (and do -p 15900:5900)
# Currently, this is required as there is not a way to set it up to auto-enter
# a match.
RUN pacman --noconfirm -S x11vnc nano
EXPOSE 5900/tcp

WORKDIR /root
ENTRYPOINT [ "sh" "./run.sh" ]
```

### Run script
```sh
#!/bin/sh
Xvfb :1 -screen 0 1024x768x24 &
# TODO: If the command is installed and then run it if its there.
x11vnc -display :1 &

DISPLAY=:1 openra-ra &

sh stream.sh
```

## Future Ideas
The thing I liked about this overall project was by shifting to something that
is mostly already made I can work implementing and solving smaller problems.

### Lichess TV Ideas
These ideas corresponding to the idea of re-streaming Lichess.

- Remove the menu bar and Previously on Lichess TV text. I originally attempted
  this with a Python script using Selenium and then again with a TamperMonkey
  script which worked well except at the end of the match where the page
  essentially reloads, they come back.
- Update the stream description to mention who is playing. This involves
  detecting when the game finishes and hit an API call on Twitch.
- Post in chat the link for the completed match so viewers can use the analysis
  tools from Lichess' website.
- Write my own tournament stream controller/narrator. Think of live streaming
  of Tennis tournaments where you are watching one match and the broadcast
  switches to another, often well know/higher seeds. This problem represents
  one that I know I will face in my own streaming project.
- Enable the game sounds - Firefox has blocked them by default and I am not
  sure how to re-enable them without a mouse and interactive session.
- Custom game renderer - Eliminate the use of Firefox.

### OpenRA Ideas
These ideas corresponding to the idea of streaming OpenRA.

- Set-up the compose configuration which starts the dedicated server and
  connects to it.
- Set-up an automated spectator which is more like a spectator AI. This is
  highly unlikely to be something that I will try to work on.

### Container

- Publish a base container image for the "Stream a web page"
- Make the Twitch end-point configurable. Currently it is set to Sydney one as
  I'm in Australia.


## Source / Pre-built image
The sources for Lichess above are available from my [lichess_streamer][6]
repository on GitHub. I may rew-work this in the future to be a twitch_streamer
repository with lichess and openra branches instead.

### Pre-built Image
* A pre-built image can be pulled from GitHub's Container Registry (ghcr.io).
* Commands are given for `docker` and `podman`.

### Set-up
If you haven't pulled from GitHub's registry then you may need to
[authenticate][5] with it. This requires you set-up a Personal Access Token 
and provide it to Docker or Podman. At the time of writing this I had to use
Classic token with the registry read scope to be able to pull the image.

```
docker login ghcr.io -u USERNAME
podman login ghcr.io -u USERNAME
```

### Single
```
docker pull ghcr.io/donno/lichess_streamer:v1
podman pull ghcr.io/donno/lichess_streamer:v1
```

[0]: https://github.com/obsproject/obs-studio/issues/8021#issuecomment-1376148786
[1]: https://obsproject.com/forum/threads/last-version-of-obs-to-support-windows-8-1.165070/post-605624
[2]: https://rufus.ie/en/
[3]: https://help.twitch.tv/s/twitch-ingest-recommendation
[4]: https://www.openra.net/
[5]: https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry#authenticating-to-the-container-registry
[6]: https://github.com/donno/lichess_streamer