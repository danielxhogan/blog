+++
title = 'Lesson 1.5: Streams'
date = 2025-06-17T10:03:17-07:00
draft = false
tutorials = 'Libav Tutorial'
+++

![dredd flames](/images/LibavTutorial/Lesson_1.5/dredd_flames.jpg)

In this lesson, you will learn how take any number of inputs with any number of
streams and dynamically select which of the input streams will be muxed into
the output file. This will be similar to using the `-map` option with `ffmpeg`.
All the code for this tutorial can be found
[here](https://github.com/danielxhogan/LibavTutorial/tree/main/Lesson%201%3A%20Remux/Lesson%201.5%3A%20Streams).

## Argument Structure

When using this command, you will choose at least one input files. Each
input file name must be preceeding by a `-i` flag. Each input will be followed
by a `-map` flag and then a sequence of number specifying the streams from that
input that will be muxed into the output file. Check the `run.sh` file for
examples. A stream can only be chosen once. The last two arguments will be used
as the output file path and the metadata title, respectively.

## InputContext struct

{{< highlight c >}}
typedef struct InputContext {
  AVFormatContext *fmt_ctx;
  const char *filename;
  int *map;
} InputContext;
{{< /highlight >}}

For each input, an InputContext struct will be created to hold information about
the input. The `map` field is a pointer to an int array. The index of each
element will correspond the index of each element in the `streams` array of the
`AVFormatContext` struct for the input, and the value will be the
index that that stream will be placed into in the output file, if that stream
was chosen by the `-map` argument. Otherwise, it will have a value of
`INACTIVE_STREAM` which is defined as `-1`.

## Allocate InputContext's

{{< highlight c >}}
for (i = 1; i < argc - 3; i++) {
  if (!(strcmp(argv[i], "-i"))) nb_inputs++;
  if (!(strcmp(argv[i], "-map"))) nb_inputs--;
}

if (nb_inputs != 0) {
  fprintf(stderr,
    "Number of inputs must match number of maps.\n");
  goto end;
}
nb_inputs = (i + 1) / 4;

if (!(inputs = av_calloc(nb_inputs, sizeof(InputContext *)))) {
  fprintf(stderr, "Failed to allocate InputContext array.\n");
  ret = AVERROR(ENOMEM);
  goto end;
}

for (i = 0; i < nb_inputs; i++) {
  if (!(inputs[i] = av_mallocz(sizeof(InputContext)))) {
    fprintf(stderr,
      "Failed to allocate Input Context for input '%d'.\n", i);
    ret = AVERROR(ENOMEM);
    goto end;
  }
}
{{< /highlight >}}

First we must check to make sure that command is formed correctly. We check to
make sure the number of `-i` arguments matches the number of `-map` arguments.
Then we calculate the number of inputs with `nb_inputs = (i + 1) / 4;`. The way
this works is that after the above for loop runs, `i` is the index of the last
instance of `-map` in `argv`. We add one to get the index of the map value and
divide by 4 because there are 4 elements in `argv` for each input - `-i`, `value`,
`-map`, `value`.

Now that we know the number of inputs, we allocate `inputs` using `av_calloc`.
Each element will be a pointer to an `InputContext` and there will be `nb_inputs`
elements in the array. Then we allocate each `InputContext` using `av_mallocz`.
Each pointer returned is assigned to the corresponding index of the `inputs` array.

## Initialize Inputs

{{< highlight c >}}
for (i = 0; i < nb_inputs; i++)
{
  if ((ret = initialize_input(inputs[i],
    &out_stream_idx, out_fmt_ctx, i, argv)) < 0)
  {
    fprintf(stderr, "Failed to initialize input '%d'.\n", i);
    goto end;
  }
}
{{< /highlight >}}

Each input is passed into `initialize_input`. The function also needs a pointer
to `out_stream_idx`. This variable keeps track of the next index each new stream
will be placed into in the output. The value gets incremented each time a stream
for an input is added to `out_fmt_ctx`. This happens inside of `initialize_input`
because there can be multiple streams per input, but we also need a reference to
the value in the `main` function so that everytime the function is called, we
know where the count left off from the previous input.

{{< highlight c >}}
int initialize_input(InputContext *input_ctx,
  int *out_stream_idx, AVFormatContext *out_fmt_ctx,
  int input_idx, char **argv)
{
  const char *map;
  int ret, i, in_stream_idx;
  AVStream *in_stream;

  map = argv[(input_idx * 4) + 4];
  input_ctx->filename = argv[(input_idx * 4) + 2];

  if ((ret = avformat_open_input(&input_ctx->fmt_ctx,
    input_ctx->filename, NULL, NULL)) < 0)
  {
    fprintf(stderr, "Failed to open input file: '%s'.\n",
      input_ctx->filename);
    return ret;
  }

  if ((ret =
    avformat_find_stream_info(input_ctx->fmt_ctx, NULL)) < 0)
  {
    fprintf(stderr,
      "Failed to retrieve input stream info file: '%s'.\n",
      input_ctx->filename);
    return ret;
  }

  if (!(input_ctx->map =
    av_calloc(input_ctx->fmt_ctx->nb_streams, sizeof(int))))
  {
    fprintf(stderr,
      "Failed to allocate map array for input '%d'.\n",
      input_idx);
    ret = AVERROR(ENOMEM);
    return ret;
  }

  for (i = 0; i < input_ctx->fmt_ctx->nb_streams; i++) {
    input_ctx->map[i] = INACTIVE_STREAM;
  }

  for (i = 0; map[i] != '\0'; i++)
  {
    in_stream_idx = map[i] - '0';
    if (in_stream_idx < 0 || in_stream_idx > 9) {
      fprintf(stderr, "Invalid character found "
        "when parsing map for input '%d'.\n", input_idx);
      return 0;
    }

    if (in_stream_idx >= input_ctx->fmt_ctx->nb_streams) {
      fprintf(stderr,
        "Stream index '%d' does not exist for input '%d'.\n",
        in_stream_idx, input_idx);
      return 0;
    }

    in_stream = input_ctx->fmt_ctx->streams[in_stream_idx];
    input_ctx->map[in_stream_idx] = *out_stream_idx;
    *out_stream_idx += 1;

    if ((ret = initialize_stream(out_fmt_ctx, in_stream,
      input_idx, in_stream_idx)) < 0)
    {
      fprintf(stderr,
        "Failed to initialize stream '%d' for input '%d'.\n",
        i, input_idx);
      return ret;
    }
  }

  return ret;
}
{{< /highlight >}}

First we use the `input_idx` to reference the `argv` array and get the `map` and
`filename` values for the current input. Then, like always we call
`avformat_open_input` and `avformat_find_stream_info`. Next we allocate the `map`
array for the `input_ctx` to have length equal to the number of streams in the
input. We initialize each value to `INACTIVE_STREAM`.

Next, we iterate through
the `map` string, which comes directly from the value passed to the `-map`
argument on the command line. Each element will be a `char` so we subtract
`'0'` which will treat the characters like their ascii code number and the
result will be the `int` version of the value passed in as long the value
was a number. That's what we check for next. As long as the value passed in was
a number between 0 and 9, the `char` to `int` conversion will be the same number.
If the user passed in some other character, then the value resulting from
subtracting that characters ascii code number from the ascii code number of `'0'`
will be outside the bounds of 0 and 9.

Next we check to make sure the stream chosen by the user actually exists in the
input. If so, we get a reference to the `AVStream` from the `streams` array that
corresponds to index selected. Then we set the value of that index in the `map`
array for the `input_ctx` to the next available output index using
`out_stream_idx`. Next, we initialize the stream.

## Initialize Stream

{{< highlight c >}}
int initialize_stream(AVFormatContext *out_fmt_ctx,
  AVStream *in_stream, int input_idx, int stream_idx)
{
  AVStream *out_stream;
  int ret = 0;

  if (!(out_stream = avformat_new_stream(out_fmt_ctx, NULL))) {
    fprintf(stderr, "Failed to allocate output stream for "
      "input '%d' stream '%d'.\n", input_idx, stream_idx);
    ret = AVERROR(ENOMEM);
    return ret;
  }

  if ((ret = avcodec_parameters_copy(out_stream->codecpar,
    in_stream->codecpar)) < 0)
  {
    fprintf(stderr, "Failed to copy codec parameters "
      "for input '%d' stream'%d'.\n", input_idx, stream_idx);
    return ret;
  }
  out_stream->codecpar->codec_tag = 0;

  if (
    (out_stream->codecpar->codec_type == AVMEDIA_TYPE_SUBTITLE) &&
    !(strcmp(out_fmt_ctx->oformat->name, "mp4")))
  {
    out_stream->codecpar->codec_id = AV_CODEC_ID_MOV_TEXT;
  }

  if ((ret = av_dict_copy(&out_stream->metadata,
    in_stream->metadata,
    AV_DICT_DONT_OVERWRITE)) < 0)
  {
    fprintf(stderr,
      "Failed to copy metadata for input '%d' stream '%d'.\n",
      input_idx, stream_idx);
    return ret;
  }
  return ret;
}
{{< /highlight >}}

The `initialize_stream` function is pretty similar to the ones from previous
examples. The first main difference is that we already know what stream we want
as opposed to previous examples where we would search for a video, audio, or
subtitle stream. we also don't care about passing out an `AVStream` or
`stream_idx`.

Once `initialize_stream` returns, `initialize_input` returns and goes to the
next input. Once the main fuction loop is finished initializing all inputs, it
continues much like in previous examples. It copies file metadata from the first
input to the output, it sets the metadata tile based on the user supplied value,
it copies chapters from the first input, it allocates a `pkt` variable, opens an
output file for writing, and writes the header to the file. Once we get to the
writing loop is where things change.

## Muxing

{{< highlight c >}}
for (i = 0; i < nb_inputs; i++) {
  while ((ret = av_read_frame(inputs[i]->fmt_ctx, pkt)) >= 0)
  {
    in_stream = inputs[i]->fmt_ctx->streams[pkt->stream_index];

    pkt->stream_index = inputs[i]->map[pkt->stream_index];

    if (pkt->stream_index == INACTIVE_STREAM) {
      av_packet_unref(pkt);
      continue;
    }

    out_stream = out_fmt_ctx->streams[pkt->stream_index];
    av_packet_rescale_ts(pkt, in_stream->time_base,
      out_stream->time_base);
    pkt->pos = -1;

    if ((ret =
      av_interleaved_write_frame(out_fmt_ctx, pkt)) < 0)
    {
      fprintf(stderr, "Failed to write packet to file "
        "for input '%d' stream '%d'.\n", i, in_stream->index);
      return ret;
    }
  }
}
{{< /highlight >}}

The first difference from previous examples is that we are looping over all the
inputs. Inside the loop we have our usual while loop that calls `av_read_frame`
and we pass in the `AVFormatContext` from the `inputs` array for the current input.
We use the `stream_index` field on the `pkt` returned to get a reference to the
input stream the packet was read from. Then we use the `map` array for the current
input to determine the index that this packet should be placed into in the output
file. If the value is `INACTIVE_STREAM`, that means it's not one of the streams
the user chose so we skip it. Like always, we rescale the timestamp,
set `pkt->pos = -1` so that the muxer will set it properly, and then write the
packet to the output file. Once  all inputs have been iterated over, we write the
trailer and free our variables.

The next lesson will be the exact opposite of this example. It will take one single
input and produce several outputs. It will split the input into individual streams
and output each stream in it's own file.

[Go To Next Lesson - 1.6: Demux](/posts/libav-tutorial/lesson-1.6-demux/)

[Go To Previous Lesson - 1.4: Add Subtitles](/posts/libav-tutorial/lesson-1.4-add-subtitles/)

[View All Lessons](/tutorials/libav-tutorial/)