+++
title = 'Lesson 1.7: Clipping'
date = 2025-07-03T22:57:12-07:00
draft = false
tutorials = 'Libav Tutorial'
+++

![alive](/images/LibavTutorial/Lesson_1.7/alive.jpg)

In this lesson, you will learn how to make clips of videos. It will be similar
to using the `-ss` and `-t` command line options with `ffmpeg`. All the code
for this tutorial can be found
[here](https://github.com/danielxhogan/LibavTutorial/tree/main/Lesson%201%3A%20Remux/Lesson%201.7%3A%20Clipping).

## Inital Setup

{{< highlight c >}}
start_sec = strtod(argv[1], NULL);
in_filename = argv[2];
duration_sec = strtod(argv[3], NULL);
out_filename = argv[4];

if ((ret =
  avformat_open_input(&in_fmt_ctx, in_filename, NULL, NULL)) < 0)
{
  fprintf(stderr, "Failed to open input video file: '%s'.\n",
    in_filename);
  goto end;
}

if ((ret = avformat_find_stream_info(in_fmt_ctx, NULL)) < 0) {
  fprintf(stderr, "Failed to retrieve input stream info.");
  goto end;
}

if ((ret = avformat_alloc_output_context2(&out_fmt_ctx,
  NULL, NULL, out_filename)))
{
  fprintf(stderr, "Failed to allocate output format context.\n");
  goto end;
}
{{< /highlight >}}

First we assign our command line arguments to our variables. The first argument
will be the time in seconds that we will seek to before we beginning reading
frames from the input file. It is like the `-ss` option in `ffmpeg`. We use the
`strtod` function to convert the input, which is a `char` by default, into a
`double`. Then, as usual, we call `avformat_open_input` on the input file to
create an `AVFormatContext` for the input and `avformat_alloc_output_context2`
with the output filename specified by the user to create an `AVFormatContext`
for the output.

## Initialize Streams

{{< highlight c >}}
for (int i = 0; i < in_fmt_ctx->nb_streams; i++) {
  if ((ret = initialize_stream(out_fmt_ctx,
    in_fmt_ctx->streams[i])) < 0)
  {
    fprintf(stderr, "Failed to initialize stream %d.\n", i);
    goto end;
  }

  if (out_fmt_ctx->streams[i]->codecpar->codec_type ==
    AVMEDIA_TYPE_VIDEO)
  {
    video_idx = i;
  }
}

if (video_idx == -1) {
  fprintf(stderr, "Failed to find video stream.\n");
  goto end;
}
{{< /highlight >}}

Now we loop through all the streams in the input and pass each one to the
`initialize_stream` function. When we find the video stream, the record the
index in `video_idx` because we use a video frame to determine when to start
reading frames from the input to the output. The output video, like all video,
must start with a keyframe. If for some reason we haven't found a video stream
after looping through all the streams, we fail.

{{< highlight c >}}
int initialize_stream(AVFormatContext *out_fmt_ctx,
  AVStream *in_stream)
{
  AVStream *out_stream;
  int ret = 0;

  if (!(out_stream = avformat_new_stream(out_fmt_ctx, NULL))) {
    fprintf(stderr,
      "Failed to allocate output stream for input stream.\n");
    ret = AVERROR(ENOMEM);
    return ret;
  }

  if ((ret = avcodec_parameters_copy(out_stream->codecpar,
    in_stream->codecpar)) < 0)
  {
    fprintf(stderr,
      "Failed to copy codec parameters for input stream.\n");
    return ret;
  }
  out_stream->codecpar->codec_tag = 0;

  if ((ret = av_dict_copy(&out_stream->metadata,
    in_stream->metadata, AV_DICT_DONT_OVERWRITE)) < 0)
  {
    fprintf(stderr,
      "Failed to copy metadata for input stream.\n");
    return ret;
  }
  return ret;
}
{{< /highlight >}}

The `initialize_stream` function is pretty standard. It takes in the input stream
that will be used to copy parameters, and the output context that the new stream
will be added to. We use `avformat_new_stream` to create the new stream and
`avcodec_parameters_copy` to copy codec parameters from the input stream to the
output stream. We set `codec_tag` to `0` so it will be set properly by the muxer.
Finally, we use `av_dict_copy` to copy stream metadata from the input stream to
the output stream.

## Seek Frame

{{< highlight c >}}
start_ts =
  start_sec *
  in_fmt_ctx->streams[video_idx]->time_base.den /
  in_fmt_ctx->streams[video_idx]->time_base.num;

if ((ret =
  av_seek_frame(in_fmt_ctx, video_idx, start_ts,
    AVSEEK_FLAG_BACKWARD)) < 0)
{
  fprintf(stderr, "Failed to seek to start frame.\n");
  goto end;
}
{{< /highlight >}}

Next, we convert the start time in seconds into a timestamp using the timebase of
the video stream. Then we call `av_seek_frame` to seek to the closest keyframe to
the timestamp.

## Make Clip

{{< highlight c >}}
while ((ret = av_read_frame(in_fmt_ctx, pkt)) >= 0)
{
  if (
    pkt->dts < 0 ||
    pkt->pts < 0
  ) {
      av_packet_unref(pkt);
      continue;
    }

  in_stream = in_fmt_ctx->streams[pkt->stream_index];
  out_stream = out_fmt_ctx->streams[pkt->stream_index];

  if (first_dts_set == NOT_SET)
  {
    if (pkt->stream_index == video_idx)
    {
      if (!(pkt->flags & AV_PKT_FLAG_KEY)) {
        av_packet_unref(pkt);
        continue;
      }

      first_dts = pkt->dts;
      first_dts_set = SET;
      duration_ts = av_rescale_q(duration_sec * AV_TIME_BASE,
        AV_TIME_BASE_Q,
        in_fmt_ctx->streams[video_idx]->time_base);
      end_ts = first_dts + duration_ts;
    }
    else {
      av_packet_unref(pkt);
      continue;
    }
  }

  if (pkt->dts > end_ts) {
    av_packet_unref(pkt);
    break;
  }

  pkt->pts = av_rescale_q(pkt->pts - first_dts,
    in_stream->time_base, out_stream->time_base);

  pkt->dts = av_rescale_q(pkt->dts - first_dts,
    in_stream->time_base, out_stream->time_base);

  pkt->duration = av_rescale_q(pkt->duration,
    in_stream->time_base,out_stream->time_base);

  pkt->pos = -1;
  if ((ret = av_interleaved_write_frame(out_fmt_ctx, pkt)) < 0) {
    fprintf(stderr, "Failed to write packet to file.\n");
    goto end;
  }

  av_packet_unref(pkt);
}
{{< /highlight >}}

Once we've allocated an `AVPacket`, opened the output file, and written the file
header, we are ready to start reading packets from the input file. If we read any
packets with negative timestampes, we ignore them. Then we check if `first_dts_set`
is `SET`. This value tells us if `first_dts` is set, which is the decode time
stamp of the frame that will be the first frame of clip. `av_seek_frame` does
not always seek perfectly to a keyframe so we need to check this ourselves
before we start reading writing frames to the output.

If `first_dts` is not set, we check if the current frame is from the video
stream beause we are looking for a video keyframe. Then we check if the current
frame is a keyframe with the `AV_PKT_FLAG_KEY` flag. If not, we go to the next
frame. If it is a keyframe, then this is the first video key frame we've read
from the input file. We set `first_dts` to the `dts` of the current frame.
We set `first_dts_set` to `SET` so we know we've found the first_dts. Then we
rescale the `duration_sec` to the timebase of the the video stream. Finally, we
calculate the `end_ts` by adding `duration_ts` to `first_dts`. This is the
timestamp that will tell us once we`ve reached the desired length for the clip.

Now that we have found the starting keyframe to start the clip from, and we know
when to end the clip, we first check if the current frame's `dts` is past
`end_ts`. If so, we just stop the loop. If not, we rescale the pkt's timestamps
to the output timebase, just like usual, except we first subtract the value of
`first_dts` from the current timestamp so that every timestamp for the entire
video will be offset by the same number and timestamps will start at `0`.

Then we set `pkt->pos` to `-1` and write the frame to the output file. Once that's
done, we write the trailer and do our cleanup and we're done. The output file
should now contain a clip of the original file that begins at the closest video
key frame to the timestamp specified by the user with a length equal to the
duration specified by the user.

The next lesson will be the start of Section 2 and we will begin learning about
transcoding video and audio.

[Go To Previous Lesson - 1.6: Demux](/posts/libav-tutorial/lesson-1.6-demux/)

[View All Lessons](/tutorials/libav-tutorial/)