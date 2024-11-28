# 8. Lab: Streaming video content over the network

## Instructions

0. Use the network and virtual machines from the previous labs.
1. Safely download any video from YouTube.
2. Use VLC to stream video content via the HTTP protocol and capture the exchanged packets with Wireshark.
3. Use VLC to stream video content via the RTP protocol using a multicast address and capture the exchanged packets with Wireshark.

## Additional information

[YouTube](https://en.wikipedia.org/wiki/YouTube) is an online platform for sharing video content and also serves as a social network.

[youtube-dl](https://github.com/ytdl-org/youtube-dl/) original program for downloading video content from the YouTube web platform.

[yt-dlp](https://github.com/yt-dlp/yt-dlp) is a program that improves and extends `youtube-dl`.

[VLC](https://www.videolan.org/vlc/) is a program for playing audio and video files and streaming protocols.

[Hypertext Transfer Protocol - HTTP](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol) is an application-layer protocol for sharing data on the Web. It was originally intended for publishing and receiving pages in [HyperText Markup Language - HTML](https://en.wikipedia.org/wiki/HTML) format.

[Uniform Resource Locator - URL](https://en.wikipedia.org/wiki/URL) predstavlja referenco spletni vir, ki določi njegovo lokacijo in način za njegovo pridobitev.

[Real-time Transport Protocol](https://en.wikipedia.org/wiki/Real-time_Transport_Protocol) is a network protocol for transmitting audio and video content over an IP network.

[Real Time Streaming Protocol](https://en.wikipedia.org/wiki/Real_Time_Streaming_Protocol) is an application layer network protocol that manages the transmission of multi-media content via an appropriate transport layer protocol.

[Real Time Control Protocol](https://en.wikipedia.org/wiki/RTP_Control_Protocol) is a network protocol that provides statistics and control of RTP connections.

[Multicast](https://en.wikipedia.org/wiki/Multicast) is a reference to a web resource that specifies its location on a computer network and a mechanism for retrieving it.

IP addresses designated for distribution:

| Network Block | Interval                    | No. of addresses |
|---------------|-----------------------------|------------------|
| 224.0.0.0/4   | 224.0.0.0 - 239.255.255.255 | 268435456        |

## Detailed instructions

### 1. Task

Let's install the `yt-dlp` program to download video content from the [YouTube](https://www.youtube.com/) web platform with [`curl`](https://linux.die.net/man/1/curl). We can also install the [`ffmpeg`](https://www.ffmpeg.org/) toolkit to convert between different media formats.

    apt install curl ffmpeg

    curl -L https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp -o /usr/local/bin/yt-dlp
    
    chmod a+rx /usr/local/bin/yt-dlp 

Now restart the terminal window. When we want to download a video, we can first check which formats are available for download.

    yt-dlp -F https://www.youtube.com/watch?v=aEvP2tqaZD4

    [youtube] Extracting URL: https://www.youtube.com/watch?v=aEvP2tqaZD4
	[youtube] aEvP2tqaZD4: Downloading webpage
	[youtube] aEvP2tqaZD4: Downloading ios player API JSON
	[youtube] aEvP2tqaZD4: Downloading mweb player API JSON
	[youtube] aEvP2tqaZD4: Downloading player b46bb280
	[youtube] aEvP2tqaZD4: Downloading m3u8 information
	[info] Available formats for aEvP2tqaZD4:
	ID      EXT   RESOLUTION FPS CH │   FILESIZE   TBR PROTO │ VCODEC          VBR ACODEC      ABR ASR MORE INFO
	────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
	sb2     mhtml 48x27        1    │                  mhtml │ images                                  storyboard
	sb1     mhtml 80x45        1    │                  mhtml │ images                                  storyboard
	sb0     mhtml 160x90       1    │                  mhtml │ images                                  storyboard
	233     mp4   audio only        │                  m3u8  │ audio only          unknown             Default
	234     mp4   audio only        │                  m3u8  │ audio only          unknown             Default
	139-drc m4a   audio only      2 │    1.10MiB   49k https │ audio only          mp4a.40.5   49k 22k low, DRC, m4a_dash
	139     m4a   audio only      2 │    1.10MiB   49k https │ audio only          mp4a.40.5   49k 22k low, m4a_dash
	140-drc m4a   audio only      2 │    2.91MiB  129k https │ audio only          mp4a.40.2  129k 44k medium, DRC, m4a_dash
	140     m4a   audio only      2 │    2.91MiB  130k https │ audio only          mp4a.40.2  130k 44k medium, m4a_dash
	269     mp4   256x144     25    │ ~  3.82MiB  170k m3u8  │ avc1.4D400C    170k video only
	160     mp4   256x144     25    │    1.56MiB   69k https │ avc1.4D400C     69k video only          144p, mp4_dash
	230     mp4   640x360     25    │ ~ 18.10MiB  803k m3u8  │ avc1.4D401E    803k video only
	134     mp4   640x360     25    │    8.59MiB  382k https │ avc1.4D401E    382k video only          360p, mp4_dash
	18      mp4   640x360     25  2 │ ≈ 11.50MiB  511k https │ avc1.42001E         mp4a.40.2       44k 360p
	605     mp4   640x360     25    │ ~ 12.89MiB  572k m3u8  │ vp09.00.21.08  572k video only
	243     webm  640x360     25    │    7.06MiB  314k https │ vp9            314k video only          360p, webm_dash
	232     mp4   1280x720    25    │ ~ 58.46MiB 2595k m3u8  │ avc1.4D401F   2595k video only
	136     mp4   1280x720    25    │   32.09MiB 1427k https │ avc1.4D401F   1427k video only          720p, mp4_dash
	270     mp4   1920x1080   25    │ ~105.60MiB 4687k m3u8  │ avc1.640028   4687k video only
	137     mp4   1920x1080   25    │   56.80MiB 2526k https │ avc1.640028   2526k video only          1080p, mp4_dash

Then we decide, for example, on the `136` option for video and `140` for audio and transfer the video to the local disk.

    yt-dlp -f 137+140 https://www.youtube.com/watch?v=aEvP2tqaZD4

    [youtube] Extracting URL: https://www.youtube.com/watch?v=aEvP2tqaZD4
	[youtube] aEvP2tqaZD4: Downloading webpage
	[youtube] aEvP2tqaZD4: Downloading ios player API JSON
	[youtube] aEvP2tqaZD4: Downloading mweb player API JSON
	[youtube] aEvP2tqaZD4: Downloading m3u8 information
	[info] aEvP2tqaZD4: Downloading 1 format(s): 137+140
	[download] Destination: Slovenian Impressions. Feel pure LOVE. [aEvP2tqaZD4].f137.mp4
	[download] 100% of   56.80MiB in 00:00:01 at 31.94MiB/s
	[download] Destination: Slovenian Impressions. Feel pure LOVE. [aEvP2tqaZD4].f140.m4a
	[download] 100% of    2.91MiB in 00:00:00 at 9.67MiB/s
	[Merger] Merging formats into "Slovenian Impressions. Feel pure LOVE. [aEvP2tqaZD4].mp4"
	Deleting original file Slovenian Impressions. Feel pure LOVE. [aEvP2tqaZD4].f140.m4a (pass -k to keep)
	Deleting original file Slovenian Impressions. Feel pure LOVE. [aEvP2tqaZD4].f137.mp4 (pass -k to keep)

### 2. Task

Let's install the `VLC` program for playing and streaming video and audio content.

    apt install vlc

Streaming in the `VLC` program can be started via the graphical wizard, which can be started via the `Media\Stream...` menu. Under the `File` tab, by pressing the `Add` button, we select the file we want to stream and then continue by pressing the `Stream` button.

![Select file to stream.](images/lab8-vlc1.png)

In the next step, we check the source for streaming, so that in the input field `Source:` is the path to the file we want to stream and in the input field `Type:` the type `file` is specified, and then we press the `Next` button.

![Specifying the source and source type for streaming.](images/lab8-vlc2.png)

Now we select the streaming protocol, for example, the `HTTP` protocol in the `New destination` dropdown menu and press the `Add` button.

![Select streaming protocol.](images/lab8-vlc3.png)

In the next step, select the network port for the selected protocol by entering port `8080` in the `Port` drop-down menu and specifying the path or URL where our stream will be located, for example `/`. We press the `Next` button to continue.

![Selecting the network port and URL for our stream.](images/lab8-vlc4.png)

The next step allows us to transcode the audio-video stream by selecting the `Activate Transcoding` option, and then in the `Profile` drop-down menu, we can choose the desired quality from the pre-defined profiles or, by clicking the key button, create our desired profile. We click on the `Next` button to continue.

![Choosing to use and transcoding settings.](images/lab8-vlc5.png)

When creating any profile for transcoding, we can choose from any supported protocols, which are classified according to their role in four tabs: encapsulation `Encapsulation`, video codec `Video codec`, audio codec `Audio codec` and subtitles `Subtitles`. We also enter the name of our profile in the `Profile name` input field and save it by clicking the `Save` button. We can now select it in the `Profile` drop-down menu.

![Creating any transcoding profile.](images/lab8-vlc7.png)

In the last window, we get a setup output in the `Generated stream output string` field, if we wanted to run our current stream from the command shell with the `vlc` command. Press the `Stream` button to start streaming.

![Creating stream.](images/lab8-vlc6.png)

Let's test the streaming by opening another instance of the `VLC` player and opening the stream by going to the `Media\Open Network Stream ...` menu, entering the URL to our stream in the `Please enter a network URL:` field, for example, `http://localhost:8080` and then pressing the `Play` button to start playing the stream. Also try opening the stream on another virtual machine by opening the `VLC` player and opening the stream at the URL `http://10.0.0.1:8080`.

![Playing network stream.](images/lab8-vlc8.png)

### 3. Task

Streaming in the `VLC` program can be started via the graphical wizard, which can be started via the `Media\Stream...` menu. Under the `File` tab, by pressing the `Add` button, we select the file we want to stream and then continue by pressing the `Stream` button.

![Select file to stream.](images/lab8-vlc1.png)

In the next step, we check the source for streaming, so that in the input field `Source:` is the path to the file we want to stream and in the input field `Type:` the type `file` is specified, and then we press the `Next` button.

![Specifying the source and source type for streaming.](images/lab8-vlc2.png)

Now we select the streaming protocol, for example, the `RTP / MPEG Transport Stream` protocol in the `New destination` drop-down menu and press the `Add` button.

![Select streaming protocol.](images/lab8-vlc9.png)

In the next step, we select the network port for the selected protocol, by entering the address for distribution on which the stream will be accessible in the `Address` input field, entering port `5004` in the `Port` drop-down menu and specifying the name of our stream in the `Stream name` input field, for example ` `. We press the button `Next` to continue.

![Choosing the network port and URL for our stream.](images/lab8-vlc10.png)

The next step allows us to transcode the audio-video stream by selecting the `Activate Transcoding` option, and then in the `Profile` drop-down menu, we can choose the desired quality from the pre-defined profiles or, by clicking the key button, create our desired profile. We click on the button `Next` to continue.

![Choosing to use and transcoding settings.](images/lab8-vlc5.png)

When creating any profile for transcoding, we can choose from any supported protocols, which are classified according to their role in four tabs: encapsulation `Encapsulation`, video codec `Video codec`, audio codec `Audio codec` and subtitles `Subtitles`. We also enter the name of our profile in the `Profile name` input field and save it by clicking the `Save` button. We can now select it in the `Profile` drop-down menu.

![Creating any transcoding profile.](images/lab8-vlc7.png)

In the last window, we get a setup output in the `Generated stream output string` field, if we wanted to run our current stream from the command shell with the `vlc` command. Press the `Stream` button to start streaming.

![Creating stream.](images/lab8-vlc11.png)

Let's test the streaming by opening another instance of the `VLC` player and opening the stream by going to the `Media\Open Network Stream ...` menu, entering the URL to our stream in the `Please enter a network URL:` field, for example, `rtp://224.0.0.1:5004` and then press the `Play` button to start playing the stream. Also try opening the stream on another virtual machine by opening the `VLC` player and opening the stream at the URL `rtp://224.0.0.1:5004`. If streaming is not working then the network does not support multicasting, you can fix this by setting both network adapters in both virtual machines that are currently set to `Internal Network` to `NAT Network`.

![Playing network stream.](images/lab8-vlc12.png)
