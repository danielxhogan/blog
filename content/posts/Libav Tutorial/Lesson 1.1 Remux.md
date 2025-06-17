+++
title = 'Lesson 1.1: Remux'
date = 2025-04-12T12:23:33-07:00
draft = false
tutorials = 'Libav Tutorial'
+++

![nope](/images/LibavTutorial/Lesson_1.1/nope.jpg)

In this lesson, you will learn how to use Libav to remux a video file. You will
learn about the `AVFormatContext` struct that is used for muxing and demuxing,
you will learn how to open an output file and prepare it for storing video and
audio streams, and you will learn how to read packets of data from an input
and write them to an output. All the code for this tutorial can found
[here](https://github.com/danielxhogan/LibavTutorial/tree/main/Lesson%201%3A%20Remux/Lesson%201.1%3A%20Remux).

To follow along with this tutorial, you'll need to have the following packages
installed:
- gcc
- ninja
- meson
- libavformat-dev
- libavcodec-dev
- libavutil-dev

The first thing you'll need to do is copy the meson.build file and create a
blank remux.c file. To generate a build configuration, open a terminal, `cd`
into the Remux directory, and run the command `meson setup build`. From this
point on, all you need to do to compile the code is `cd` into the build
directory and run `ninja`. The commands to run the program can be found in the
`run.sh` file. There are several video files provided in the `videos/inputs`
folder and the commands are designed to store the ouput files in
`videos/outputs` so you'll need to create that folder.

## Opening Input File & Initializing AVFormatContext

{{< highlight c >}}
AVFormatContext *in_fmt_ctx = NULL, *out_fmt_ctx = NULL;

if ((ret = avformat_open_input(&in_fmt_ctx, in_filename,
  NULL, NULL)) < 0)
{
  fprintf(stderr, "Failed to open input video file: '%s'.\n",
    in_filename);
  goto end;
}

if ((ret = avformat_find_stream_info(in_fmt_ctx, NULL)) < 0) {
  fprintf(stderr, "Failed to retrieve input stream info.");
  goto end;
}

av_dump_format(in_fmt_ctx, 0, in_filename, 0);

if ((ret = avformat_alloc_output_context2(&out_fmt_ctx,
  NULL, NULL, out_filename)))
{
  fprintf(stderr, "Failed to allocate output format context.\n");
  goto end;
}

...

end:
  avformat_close_input(&in_fmt_ctx);
  avformat_free_context(out_fmt_ctx);
{{< /highlight >}}

Once we've added our `#include` statements, declared all the necessary
variables, and defined the usage print statement, the first thing we'll do is
open the input file that is passed on the command line. The
`AVFormatContext` struct is the main struct used in Libav to represent a
file. There are two main ways for allocating an `AVFormatContext` struct,
depending on whether the file is an existing file being read from disk or
whether we are creating the file. If we are opening an existing file, we
use `avformat_open_input` and if we are creating the file, we use
`avformat_alloc_output_context2`. The former is usually paired with the
`avformat_find_stream_info` function. For this example, this function is mostly
needed if the input file is a webm or flv file. We can also call the
`av_dump_format` function which will print out information about the input file,
much like when running `ffprobe` on the file. The latter creates a blank struct
that will be manually populated.

Both functions have the option to take in a format
struct, either `AVFormatInput` or `AVFormatOutput`, that will determine
the format of the file. When opening a file, if the file does not match the
specified format, this will cause an error. If a format is not provided, it
will be auto detected. When creating a file, if a format is not specified,
the name of the output file provided will be used to determine the outuput
format, just like when using `ffmpeg`.

## Adding Streams

{{< highlight c >}}
if ((ret = v_stream_idx = av_find_best_stream(in_fmt_ctx,
  AVMEDIA_TYPE_VIDEO, -1, -1, NULL, 0)) < 0)
{
  fprintf(stderr, "Failed to find video stream in input file.\n");
  goto end;
}

if (!(out_stream = avformat_new_stream(out_fmt_ctx, NULL))) {
  fprintf(stderr, "Failed to allocate video output stream.\n");
  ret = AVERROR(ENOMEM);
  goto end;
}

if ((ret = avcodec_parameters_copy(out_stream->codecpar,
  in_fmt_ctx->streams[v_stream_idx]->codecpar)) < 0)
{
  fprintf(stderr, "failed to copy video codec parameters\n");
  goto end;
}

out_stream->codecpar->codec_tag = 0;
{{< /highlight >}}

Now that we've created a blank `AVFormatContext` struct for our output file, we
need to start populating it. The first thing we'll do is create an `AVStream`
for the video stream. To do this, we'll use the `avformat_new_stream` function,
and pass in `out_fmt_ctx`. The `AVFormatContext` struct contains a `streams`
array which contains all the streams for the file. The `avformat_new_stream`
function will allocate a new `AVStream`, add it to the streams array of
`out_fmt_ctx`, and return a pointer to it, or `NULL` on error.

Once we have a new blank stream for the video, we'll need to populate it
with all the information about the stream we want to create. In this case, we
are copying the stream from the input so we'll copy all the same information.
To do this, we'll need to know which stream in the input is the video stream.
We'll use the `av_find_best_stream` function to get the index of the `streams`
array from `in_fmt_ctx` that holds the video stream. We'll pass in
`AVMEDIA_TYPE_VIDEO` so the function will look for a video stream. It will
generally return the first video stream it finds, as long as there is a decoder
for that stream's codec. This is similar to using `ffmpeg` without using the
`map` option.

Once we know the index of the video stream, we'll use the
`avcodec_copy_parameters` function to copy the `codecpar` struct from the video
stream in `in_fmt_ctx` to the new `AVStream` we just created. Although we are
not decoding any data in this example, the output file will need to contain all
this information so that it can be decoded properly for playback or further
processing. The last thing we'll do is make sure that `codec_tag` is set to
`0`. This is to make sure it is set properly later on. This value should be set
by `avformat_write_header`, which will be called later, but the value will only
by overwritten if it is initially set to `0`.

The process for creating the audio stream is mostly the same as for the video
except we'll specify `AVMEDIA_TYPE_AUDIO` as the stream type to look for in the
input file.

## Pre Muxing

{{< highlight c >}}
  if (!(pkt = av_packet_alloc())) {
    fprintf(stderr, "Failed to allocate AVPacket.\n");
    ret = AVERROR(ENOMEM);
    goto end;
  }

  if (!(out_fmt_ctx->oformat->flags & AVFMT_NOFILE)) {
    if ((ret = avio_open(&out_fmt_ctx->pb, out_filename,
      AVIO_FLAG_WRITE)) < 0)
    {
      fprintf(stderr, "Failed to open output file.\n");
      goto end;
    }
  }

  if ((ret = avformat_write_header(out_fmt_ctx, NULL)) < 0) {
    fprintf(stderr, "Failed to write header for output file.\n");
    goto end;
  }

  ...

  end:
    ...
    av_packet_free(&pkt);
    if (out_fmt_ctx && !(out_fmt_ctx->flags & AVFMT_NOFILE))
      avio_closep(&out_fmt_ctx->pb);
    ...
{{< /highlight >}}

Now that we've prepared the output context, we have a few things to do before
we'll start muxing. The first thing we'll do is allocate an `AVPacket` that
will be used to store data read from the input file. The next thing we'll do is
create an output file that the data will be read to. Some demuxers will create
the file automatically so we first need to check if there is an `AVFMT_NOFILE`
flag on output format. If not, we call `avio_open` which will create a file
with the name passed in on the command line, open it for writing, and store a
reference to it in the `pb` field of `out_fmt_ctx`. Lastly, well call
`avformat_write_header`. This will write all the necessary metadata to the
file such as the container format, number of streams, and codecs used,
as well as any other information that will be needed to properly open and read
the contents of the file. We will also need to add lines to the `end` procedure
to free the `AVPacket` and close the file.

## Muxing

{{< highlight c >}}
  while ((ret = av_read_frame(in_fmt_ctx, pkt)) >= 0)
  {
    in_stream = in_fmt_ctx->streams[pkt->stream_index];

    if (pkt->stream_index == v_stream_idx) {
      pkt->stream_index = VIDEO_STREAM_IDX;
    }
    else if (pkt->stream_index == a_stream_idx) {
      pkt->stream_index = AUDIO_STREAM_IDX;
    }
    else {
      av_packet_unref(pkt);
      continue;
    }

    pkt->pos = -1;
    out_stream = out_fmt_ctx->streams[pkt->stream_index];
    av_packet_rescale_ts(pkt, in_stream->time_base,
      out_stream->time_base);

    if ((ret = av_interleaved_write_frame(out_fmt_ctx, pkt)) < 0)
    {
      fprintf(stderr, "Failed to write packet to file.\n");
      goto end;
    }

    av_packet_unref(pkt);

    if ((ret = av_write_trailer(out_fmt_ctx)) < 0) {
      fprintf(stderr, "Failed to write trailer to file.\n");
      goto end;
    }
  }
{{< /highlight >}}

We'll start by calling `av_read_frame` inside a loop that runs as long the
return value is not negative. A negative value signals the end of the file,
or an error. The packet data read from the input will be stored in `pkt`. This
struct will have a field `stream_index` specifying which stream from the input
it came from. This value is used to get a reference to that stream from
`in_fmt_ctx->streams`. We check if the the packet came from the video stream or
the audio stream, and then overwrite the value of the packet's `stream_index`
because this value will be used by the muxer to determine which stream to place
the packet into in the output file. If the packet does not belong the either the
video or audio stream that was found earlier when we called
`av_find_best_stream`, it is ignored.

Next we'll set `pkt->pos` to -1 so that it will be set by the muxer. Initially
this value will indicate the position of the packet in the input file but it
needs to be reset to the position of the packet in the output file. After that,
we'll call `av_packet_rescale_ts`. This will make sure that if the output file
has a different time base for it's timestamps, the timestamp of each packet
will be converted.

Finally, we will call `av_interleaved_write_frame` to write the packet to the
output file. Once the loop has finished, all data from the input should now
be written to the new output file. The only thing left to do is call
`av_write_trailer` which will write any necessary metadata to file that wasn't
known before the writing process such as byte locations of I frames used for
seeking.

In the next lesson, we will build off the concepts in this lesson by learning
how to access, transfer, and set user defined metadata values. You will be
able to copy all metadata from the input to the output and pass in a title
value on the command line that will be used to set the metadata title of the
output. You will also learn how to transfer chapter information.

[Go To Next Lesson - 1.2: Copy Metadata](/posts/libav-tutorial/lesson-1.2-copy-metadata/)

[Go To Previous Lesson - 0: Libav Tutorial Introduction](/posts/libav-tutorial/lesson-0/)

[View All Lessons](/tutorials/libav-tutorial/)