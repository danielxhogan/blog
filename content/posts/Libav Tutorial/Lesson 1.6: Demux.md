+++
title = 'Lesson 1.6: Demux'
date = 2025-06-28T10:18:54-07:00
draft = false
tutorials = 'Libav Tutorial'
+++

![sever ders](/images/LibavTutorial/Lesson_1.6/seven_ders.jpg)

In this lesson, you will learn how to output multiple files. You will take in a
video file, loop through all the streams, and output each stream into it's own
file. All the code for this tutorial can be found
[here](https://github.com/danielxhogan/LibavTutorial/tree/main/Lesson%201%3A%20Remux/Lesson%201.6%3A%20Demux).

## Get basename

{{< highlight c >}}
in_filename = argv[1];
output_dir = argv[2];

if ((ret = avformat_open_input(&fmt_ctx, in_filename, 0, 0)) < 0)
{
  fprintf(stderr, "Failed to open input file '%s'.\n",
    in_filename);
  goto end;
}

if ((ret = avformat_find_stream_info(fmt_ctx, 0)) < 0) {
  fprintf(stderr, "Failed to retrieve input stream info.");
  goto end;
}

slash = strrchr(in_filename, '/');
if (slash) in_filename = ++slash;

if ((ret = get_len_basename(&len_basename, in_filename)) < 0) {
  printf("Failed to get length of input file basename.\n");
  goto end;
}

if (!(basename = av_mallocz((len_basename + 1) * sizeof(char)))) {
  printf("Failed to allocate memory for basename.\n");
  ret = AVERROR(ENOMEM);
  goto end;
}
strncpy(basename, in_filename, len_basename);
{{< /highlight >}}

This command takes in the path to a video a file and the path to a directory
where the output files will be stored. The name of each file will have the same
base name as the input file with an extension that matches the codec of that
stream. Each file will use the matroska format.

In the code, we first store the command line input values in `in_filename` and
`output_dir`. Then, like always, we use `avformat_open_input` and
`avformat_find_stream_info` to open the input file get info about it. Next, we
parse the `in_filename` string to get the `basename` of the input file. We use
`strrchr` to find the last instance of the character `/` in case the value is
a path. If no instance of the character is found, `strrchr` will return `null`.
Otherwise, `slash` will contain a pointer that points to the last instance of
the character. We check if it's null and if not, we set `in_filename` to point
to the next sport in memory after where `slash` points to, which will be the
first character of the base name of the input file.

{{< highlight c >}}
int get_len_basename(size_t *len_basename, const char *filename)
{
  const char *slash;
  const char *dot;
  const char *end;
  size_t filename_length;
  size_t ext_length;

  slash = strrchr(filename, '/');
  if (slash) filename = ++slash;

  dot = strrchr(filename, '.');
  if (!dot || dot == filename) {
    fprintf(stderr, "Invalid file name.\n");
    return -1;
  }

  for (end = filename; *end; end++);
  filename_length = end - filename + 1;

  for (end = dot; *end; end++);
  ext_length = end - dot + 1;

  *len_basename = filename_length - ext_length;
  return 0;
}
{{< /highlight >}}

Now that we know where the base name starts, we need to figure out where it
ends. In order to know that we need to know how long the base name is so we
use the `get_len_basename` function. First, we find the location of the `.`.
There must be a `.` present in the filename and it can't be the first
character in the `filename`. Then we find the end of the file and subtract
the beginning to get the length of the entire `filename`. Then we subtract
`dot` from the end to get the length of the extension. Then we subtract
`ext_length` from `filename_length` and that gives us the length of the
`basename`.

Back in the main fuction, now that we know how long the base name is, we can
use the `av_mallocz` function to allocate the `basename` variable and read
`len_basename` number of characters from `in_filename`.

## Initialize Streams

{{< highlight c >}}
typedef struct StreamContext {
  char *filename;
  AVFormatContext *fmt_ctx;
  AVStream *in_stream, *out_stream;
  int stream_idx;
} StreamContext;
{{< /highlight >}}

{{< highlight c >}}
// in main
if (!(streams = av_calloc(fmt_ctx->nb_streams,
sizeof(StreamContext *))))
{
  fprintf(stderr,
    "Could not allocate memory for streams array.\n");
  ret = AVERROR(ENOMEM);
  goto end;
}

// av_dump_format(fmt_ctx, 0, in_filename, 0);

for (i = 0; i < fmt_ctx->nb_streams; i++)
{
  if (!(streams[i] = av_mallocz(sizeof(StreamContext)))) {
    fprintf(stderr, "Could not allocate memory "
      "for streams for stream '%d'.\n", i);
    ret = AVERROR(ENOMEM);
    goto end;
  }

  streams[i]->filename = NULL;
  streams[i]->stream_idx = i;
  if ((ret =
    init_stream(streams[i], fmt_ctx, output_dir, basename)) < 0)
  {
    goto end;
  }
}
{{< /highlight >}}

We define a `StreamContext` struct that will contain information about each
stream read from the input file. In the main function, after getting the
`basename`, we allocate the `streams` array will will contain a pointer to a
`StreamContext` struct for each stream in the `fmt_ctx->nb_streams`. Next, we
allocate the `StreamContext`s. We initialize the `filename` to be `null` and
the `stream_idx` field will be the same as the stream index of the
`fmt_ctx->streams` array that this each `StreamContext` will be copying data
from. Next we call the `init_stream` function to initialize the
`AVFormatContext` for the output of the current stream.

{{< highlight c >}}
int init_stream(StreamContext *stream_ctx,
AVFormatContext *fmt_ctx, const char *output_dir,
const char *basename)
{
  int ret;
  const char *ext, *title;

  stream_ctx->in_stream =
    fmt_ctx->streams[stream_ctx->stream_idx];

  ext =
    avcodec_get_name(stream_ctx->in_stream->codecpar->codec_id);

  if ((ret = make_output_filename(stream_ctx, output_dir,
    basename, ext)) != 0)
  {
    fprintf(stderr, "Failed to generate output filename.\n");
    return ret;
  }

  if ((ret = avformat_alloc_output_context2(&stream_ctx->fmt_ctx,
  NULL, "matroska", stream_ctx->filename)))
  {
    fprintf(stderr,
      "Failed to allocate format context for stream %d\n",
      stream_ctx->stream_idx);
    return ret;
  }

  if ((ret = av_dict_copy(&stream_ctx->fmt_ctx->metadata,
    fmt_ctx->metadata, AV_DICT_DONT_OVERWRITE)) < 0)
  {
    fprintf(stderr, "Failed to copy file metadata.\n");
    return ret;
  }

  title = strrchr(stream_ctx->filename, '/');
  if (title) title = ++title;

  if ((ret = av_dict_set(&stream_ctx->fmt_ctx->metadata,
    "title", title, 0)) < 0)
  {
    fprintf(stderr,
    "Failed to set title for output format context.\n");
    return ret;
  }

  if ((ret = copy_chapters(stream_ctx->fmt_ctx, fmt_ctx)) < 0) {
    fprintf(stderr, "Failed to copy chapters.\n");
    return ret;
  }

  if (!(stream_ctx->out_stream =
    avformat_new_stream(stream_ctx->fmt_ctx, NULL)))
  {
    fprintf(stderr,
      "Failed to allocate new output stream for stream %d\n",
      stream_ctx->stream_idx);
    ret = AVERROR(ENOMEM);
    return ret;
  }

  if ((ret =
    avcodec_parameters_copy(stream_ctx->out_stream->codecpar,
      stream_ctx->in_stream->codecpar)) < 0)
  {
    fprintf(stderr, "Failed to copy codec params for stream %d\n",
      stream_ctx->stream_idx);
    return ret;
  }

  if ((ret = av_dict_copy(&stream_ctx->out_stream->metadata,
    stream_ctx->in_stream->metadata, AV_DICT_DONT_OVERWRITE)) < 0)
  {
    fprintf(stderr, "Failed to copy stream metadata.\n");
    return ret;
  }

  if (!(stream_ctx->fmt_ctx->flags & AVFMT_NOFILE)) {
    if ((ret = avio_open(&stream_ctx->fmt_ctx->pb,
      stream_ctx->filename, AVIO_FLAG_WRITE)) < 0)
    {
      fprintf(stderr, "Failed to open file for stream %d\n",
        stream_ctx->stream_idx);
      return ret;
    }
  }

  if ((ret =
    avformat_write_header(stream_ctx->fmt_ctx, NULL)) < 0)
  {
    fprintf(stderr,
      "Error writing header to file for stream %d\n",
      stream_ctx->stream_idx);
    return ret;
  }

  return 0;
}
{{< /highlight >}}

First, we get a reference to the `AVStream` from `fmt_ctx->streams` that the
output stream being initialized will be copying from. Next, we use the
`avcodec_get_name` function to get the name of the codec that was used for the
current stream as a string and will be used as the extension for the output
file. Then we call the `make_output_filename` function which concatenates the
`output_dir` specified on the command line with the `basename` and the `ext`.

Next, we allocate an `AVFormatContext` for the output of the current stream. We
specify the `matroska` format because otherwise `avformat_alloc_output_context2`
will try to guess the format from the extension. Then we copy all metadata from
the input file to the currenct output. Next, we will set the metadata tile for
the output file using `strrchr` to exclude everything before the last `/`. That
way the title will just be the filename and not the entire output directory. Then
we copy chapters. Then we create an `AVStream` for the output stream and copy
codec parameters from the input stream to the output stream. Then we copy stream
metadata from the input stream to the output stream. Then we open a file that
will have the output stream written to and write the header. At this point, the
output stream is all ready to start receiving data from the input file.

## Copy Streams

{{< highlight c >}}
if (!(pkt = av_packet_alloc())) {
  fprintf(stderr, "Failed to allocate AVPacket.\n");
  goto end;
}

while ((ret = av_read_frame(fmt_ctx, pkt)) >= 0)
{
  stream_idx = pkt->stream_index;
  pkt->stream_index = 0;
  av_packet_rescale_ts(pkt,
    streams[stream_idx]->in_stream->time_base,
    streams[stream_idx]->out_stream->time_base);

  if ((ret =
    av_interleaved_write_frame(streams[stream_idx]->fmt_ctx,
    pkt)) < 0)
  {
    fprintf(stderr,
      "Error writing packet to file for stream %d\n",
      stream_idx);
    goto end;
  }
  av_packet_unref(pkt);
}

for (i = 0; i < fmt_ctx->nb_streams; i++)
{
  if ((ret = av_write_trailer(streams[i]->fmt_ctx)) < 0) {
    fprintf(stderr, "Error writing trailer for stream %d\n",
      streams[i]->stream_idx);
    goto end;
  }
}

for (i = 0; i < fmt_ctx->nb_streams; i++)
{
  if ((ret = av_write_trailer(streams[i]->fmt_ctx)) < 0) {
    fprintf(stderr, "Error writing trailer for stream %d\n",
      streams[i]->stream_idx);
    goto end;
  }
}
{{< /highlight >}}

Once we've looped through all the streams in the input file and initialized an
output context, stream, and file for each, we can start reading data. Like
always, we run a while loop that call `av_read_frame`. We store the original
`pkt->stream_index` value in `stream_idx` and then set it to `0` because each
stream is getting it's own output file so each stream will be the only stream
in the output. We use the `stream_idx` to get a reference to the `StreamContext`
for output stream that the current `pkt` will be written to and use the
`in_stream` and `out_stream` references to rescale the timestamps. Then we use
the `fmt_ctx` reference to write the packet to the output file.

Once we're done reading all data, we write trailers to all of the output files
and that's it. Each stream from the input file will now be in it's own file.

In the next lesson we will learn how to make clips of videos. It will be similar
to using the `-ss` and `-t` command line options of `ffmpeg`. We will use the
`av_seek_frame` function to seek to a point in the input file and use the
`AV_PKT_FLAG_KEY` flag to make sure that our clips always begin with a keyframe
so they can be properly decoded. We will adjust the timestamps of each frame
to make sure timestamps of our video start at 0 and we will check the timestamp
value to know when to stop reading from the input to get the desired length
for the clip.

[Go To Next Lesson - 1.7: Clipping](/posts/libav-tutorial/lesson-1.7-clipping/)

[Go To Previous Lesson - 1.5: Streams](/posts/libav-tutorial/lesson-1.5-streams/)

[View All Lessons](/tutorials/libav-tutorial/)