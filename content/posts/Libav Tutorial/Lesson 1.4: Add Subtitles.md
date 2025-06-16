+++
title = 'Lesson 1.4: Add Subtitles'
date = 2025-06-13T22:59:16-07:00
draft = false
tutorials = 'Libav Tutorial'
+++

![on the hunt](/images/LibavTutorial/Lesson_1.4/on_the_hunt.jpg)

In this lesson, you will learn how to embed subtitles stored in a text file into
a video file. This is similar to the example in the previous lesson example the
data in the file will be interpreted as a stream, not as metadata. As with the
previous lesson, there are test videos in the `videos/inputs` folder and test
commands in the `run.sh` file. Running these commands will embed the subtitle
information for `subtitles.vtt` into the output file as a subtitle stream. The
`vtt` format is only supported in the `mkv`, `mp4`, and `webm` formats. All the
code for this tutorial can be found
[here](https://github.com/danielxhogan/LibavTutorial/tree/main/Lesson%201%3A%20Remux/Lesson%201.4%3A%20Add%20Subtitles).

## Subtitle File Structure

{{< highlight c >}}
WEBVTT

00:00:07.549 --> 00:00:09.885
  <u><i>I know kung fu</i></u>

00:00:10.844 --> 00:00:11.720
  <b>Show me</b>
{{< /highlight >}}

The file should always start with `WEBVTT` to indicate to any software reading
the file that it contains `webvtt` subtitles. However, `avformat_open_input` is
smart enough to know these are `webvtt` subtitles if the file extension is
`.vtt`. Even if it has the extension `.txt`, you can still specify that the file
is `webvtt` by using `av_find_input_format("webvtt")` and passing the
`AVInputFormat` returned to `avformat_open_input`.

Each subtitle starts with a timestamp in the format `HH:MM:SS.sss`. The start
timestamp is listed first, followed by the `-->` symbol, followed by the end
timestamp. Any lines of text following the timestamps will be displayed on
screen between these timestamps. A blank line indicates the end of that
subtitle. Some basic `html` tags are supported in `mkv` and `webm`. For more
information about the construction of `webvtt` files, check out the
[spec](https://www.w3.org/TR/webvtt1).

## Opening Subtitle File

{{< highlight c >}}
const char *v_filename, *s_filename, *out_filename, *title;

AVFormatContext *v_fmt_ctx = NULL, *s_fmt_ctx = NULL,
  *out_fmt_ctx = NULL;

const AVInputFormat *s_fmt;

...

s_filename = argv[2];

...

if (!(s_fmt = av_find_input_format("webvtt"))) {
  fprintf(stderr, "Failed to find format for subtitle file.\n");
  ret = AVERROR_UNKNOWN;
  goto end;
}

if ((ret = avformat_open_input(&s_fmt_ctx, s_filename,
  s_fmt, NULL)) < 0)

  fprintf(stderr, "Failed to open subtitle file: '%s'.\n",
    s_filename);
  goto end;
}
{{< /highlight >}}

Opening a subtitle is almost identical to opening a chapter file from the last
example. The only difference is that we are specifying the `webvtt` format to
`av_find_input_format`. As explained above, this is only necessary the subtitle
file doesn't have a `.vtt` extension and doesn't start with `WEBVTT`. Otherwise
we could just pass `NULL` to `avformat_open_input` for the format.

## Writing Subtitle Data

{{< highlight c >}}
while ((ret = av_read_frame(s_fmt_ctx, pkt)) >= 0)
{
  in_stream = s_fmt_ctx->streams[pkt->stream_index];
  out_stream = out_fmt_ctx->streams[SUBTITLE_STREAM_IDX];

  av_packet_rescale_ts(pkt, in_stream->time_base,
    out_stream->time_base);
  pkt->stream_index = SUBTITLE_STREAM_IDX;
  pkt->pos = -1;

  if ((ret = av_interleaved_write_frame(out_fmt_ctx, pkt)) < 0) {
    fprintf(stderr, "Failed to write subtitle packet to file.\n");
    goto end;
  }

  av_packet_unref(pkt);
}
{{< /highlight >}}

Unlike chapters, subtitles are not stored as metadata, they stored as a stream,
similar to how audio and video data are stored. This means to copy subtitles we
need to run a loop that calls `av_read_frame`, just like we do to copy the
video and audio. We pass in `s_fmt_ctx` to read frames from the subtitle file
and they are written into the stream specified by `SUBTITLE_STREAM_IDX` in the
output file.

In the next lesson you will learn how to take in multiple video files and
dynamically choose which streams to include in the output based on user input.
This will be similar to how the `-map` option works in `ffmpeg`.

[Go To Previous Lesson - 1.3: Add Chapters](/posts/libav-tutorial/lesson-1.3-add-chapters/)

[View All Lessons](/tutorials/libav-tutorial/)
