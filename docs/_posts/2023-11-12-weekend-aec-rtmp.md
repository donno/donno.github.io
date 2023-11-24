---
layout: post
title:  "Weekend - AEC and RTMP"
date:   2023-11-12 22:00:00 +0930
---

This was quite the busy weekend for me and included two programming projects.
The first was parsing AEC Media Feed data for Australia's 2023 Referendum and
the second was looking at the RTMP and FLV protocol and format respectively.

## AEC Data Project
The first project started Friday evening and involved working with the
[AEC Media Feed][0] data for Australia's 2023 Referendum. For those unaware,
within Australia a referendum is a vote to change  Australia's Constitution. In
2023, the proposed change had two parts, the first was recognising the First
Peoples of Australia and the  second was establishing a group to represent
these people directly to parliament on issues related to themselves nd granting
power to make  regarding that group. However, this was covered by a single
question of to include both parts or not.

From a technical standpoint this referendum is a lot simpler than a federal
election. This is because there are only two choices (Yes or No) rather than
which member from a given party to elect to the senate or the house.

In this case I used the verbose and detailed media feed for 29581 (the code for
the 2023 Referendum). This differences from the typical election in that the
schema for the data uses the AEC media feed schema rather than the EML
(Election Mark-up Language) [schema][1].

The information stored in the file for a given time is as follows:
* Contests - this has a breakdown per polling districts
    * The number of enrolments.
    * The number of historical enrolments - I assume this means since the last
      election.
    * The number of votes (and their percentages) for the options, which in
      this case is simply Yes or No.
        * These are broken down by vote type into ordinary, absent,
          provisional, pre-poll and postal.
    * The number of formal, informal and totals including the breakdown
       of vote type as above.
    * The polling district is divided further by polling places.
* Analysis  - this is done by-state, so for each state there is:
    * The number of enrolments.
    * The number of historical enrolments.
    * The number of votes (and their percentages) for the options, which in this
      case is simply Yes or No.
        * These are broken down by vote type into ordinary, absent, provisional,
           pre-poll and postal.
    ** The number of formal, informal and totals including the breakdown
       of vote type as above.

For simplicity, in the data format they consider the territories (ACT and NT)
as states.

Essentially, I wrote a Python script to parse the information by state and
by district. I however got side-tracked by the next project that I didn't get
to the point where I could generate either graphs or diagrams let alone a
time-lapse of the data as I intended.

## RTMP / FLV Project

I took a detour looking into RTMP (Real-Time Messaging Protocol), which handles
video and audio streaming using technology developed by Macromedia/Adobe
made for Flash.

The first step was setting up a RTMP server so I had somewhere to stream video
to that wasn't straight to Twitch, which does act as a RTMP server.

### Set-up RTMP Server
This was done by following the guide of using [Nginx-RTMP on Ubuntu][4]. The
quick version is, I had Ubuntu 22.04 in WSL2.
```
apt install libnginx-mod-rtmp
nano /etc/nginx/nginx.conf
# <add the configuration fragmnet below. >
systemctl reload nginx.service
```
Added the following to the nginx.conf
```
rtmp {
        server {
                listen 1935;
                chunk_size 4096;
                allow publish 127.0.0.1;
                allow publish <ip of Windows machine>;
                deny publish all;
                application live {
                        live on;
                        record off;
                }
        }
}
```

To test that it worked I configured `OBS Studio` to stream to this server.
I used `ffplay` to watch the stream. This took very little time to set-up.

### Python Sample
Starting off I came across [python-librtmp][5] which provides a thin Python
layer over `librtmp` which does the heavy lifting for RTMP.

The basic script is:
```python
import librtmp

def serve_video():
    connection = librtmp.RTMP("rtmp://172.25.232.245/live/python_stream",
                              live=True)
    connection.connect()

    stream = connection.create_stream(writeable=True)

    for video_data in video_data_generator():
        # The following is a tiny wrapper over RTMP_Write().
        result = stream.write(video_data)

    stream.close()

if __name__ == '__main__':
    serve_video()
```

The challenge here is what video_data should be. The Python library documents it
as being bytes that contain the FLV data to write to the stream. The data must
can contain multiple FLV tags but must contain complete tags.

Stepping inside `RTMP_Write()` the requirements seem to be data formatted using
the FLV file format. The signature at the start of that format is the bytes FLV
which is what that function checks for at the start of the given buffer.
The function then skips forward 13 bytes which is past the header (9-bytes)
and the previous tag size which will be 0 for the first tag.

The FLV file format is documented in the [F4V format specific][6], however
that documents another format, F4V as well. The relevant part is in
Annex E of version 10.1 of the specification.

From there I started trying to implement the FLV header and a file body including
tag which ends up including the video data and H263VIDEOPACKET.

The H263VIDEOPACKET is where I hit a bit of a wall. My attempt to generate it
involved using [PyAV][7], such that it handled encoding an image with the
relevant H.263 codec. The snippet essentially came down to this:
```python
import libav
import numpy as np

def generate_packets():
    codec = av.Codec('flv', 'w')

    encoder = codec.create()
    encoder.width = 1408
    encoder.height = 1152
    encoder.pix_fmt = 'yuv420p'

    for frame_i in range(total_frames):
        img = np.empty((1408, 1152, 3))
        img[:, :, 0] = 0.5 + 0.5 * np.sin(2 * np.pi * (0 / 3 + frame_i / total_frames))
        img[:, :, 1] = 0.5 + 0.5 * np.sin(2 * np.pi * (1 / 3 + frame_i / total_frames))
        img[:, :, 2] = 0.5 + 0.5 * np.sin(2 * np.pi * (2 / 3 + frame_i / total_frames))

        img = np.round(255 * img).astype(np.uint8)
        img = np.clip(img, 0, 255)

        frame = av.VideoFrame.from_ndarray(img, format="rgb24")

        packets = encoder.encode(frame)
        assert len(packets) == 1
        yield packets[0]
```

I was confident that most this working as opening a FLV file and writing out a
FLV stream to it worked. That said, I didn't try to use the same principal here
of using the packet from the codec.

If I tried picking this up again then the aspect of this that I need to
validate is that the FLV file format written is correct and check
`RTMP_Write()` is pulling out the data that I'm expecting and overall wrapping
that all up.

### Conclusion
The conclusion of this project is was interesting thing to try by ultimately a
waste of time. It would have been great if I got it working without so much
hassle and to also show others how to achieve it.

The more practical solution is to use:
- OBS Studio to capture a window and stream to a RTMP Server.;
- ffmpeg to capture window and stream to RTMP server.

### Alternative with ffmpeg
This is specific to Microsoft Windows. For capturing a window there is the
gdigrab device. This almost works for simple windows but does not work for
complex ones with OpenGL or DirectX.

The following demonstrates capturing Notepad and playing it back without sending
it to a RTMP server. In this case, the scroll bar and status bar shows but not
the text area which is all black.
```
notepad.exe
ffplay.exe -f gdigrab -i title="Untitled - Notepad"
```

Capture top left corner of the screen
```
ffplay -f gdigrab -framerate 30 -offset_x 0 -offset_y 0 -video_size 800x600 -show_region 1 -i desktop
```
The show_region flag causes a box to be drawn over the top of the screen where
it is being drawn.

Send the full desktop to RTMP:
```
ffmpeg -f gdigrab -i desktop -vf scale=1280:960 -vcodec libx264 -profile:v baseline -pix_fmt yuv420p -f flv rtmp://localhost/live/test
```
The former set-up the input device to be GDI grab and to use the desktop the
rest transcode the output to FLV for the RTMP feed.

[0]: https://www.aec.gov.au/media/mediafeed/
[1]: https://docs.oasis-open.org/election/eml/v5.0/cs01/EML-Schema-Descriptions-v5.0.html
[4]: https://www.digitalocean.com/community/tutorials/how-to-set-up-a-video-streaming-server-using-nginx-rtmp-on-ubuntu-20-04
[5]: https://pythonhosted.org/python-librtmp/#streaming
[6]: http://download.macromedia.com/f4v/video_file_format_spec_v10_1.pdf
[7]: https://pyav.org/