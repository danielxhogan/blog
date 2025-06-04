+++
title = 'Lesson 1.2: Copy Metadata'
date = 2025-06-03T10:38:45-07:00
draft = false
tutorials = 'Libav Tutorial'
+++

![flat terminator](/images/LibavTutorial/Lesson_1.2/flat_terminator.jpg)

In this lesson, you will learn how to copy metadata and chapter information
from the input to the output. You will also learn about what types of metadata
are supported in different file formats and whether they support chapters or
not. As with the previous lesson, there are test videos in the `videos/inputs`
folder and depending on whether the format supports it they may have file
metadata, stream metadata, or chapter data. Commands for remuxing
these files can be found in the `run.sh` file with comments about what data
is able to be successfully copied. All the code for this tutorial can be found
[here](https://github.com/danielxhogan/LibavTutorial/tree/main/Lesson%201%3A%20Remux/Lesson%201.2%3A%20Copy%20Metadata).

## File Metadata

{{< highlight c >}}
// in main
const char *in_filename, *out_filename, *title;

...

title = argv[3];

...

if ((ret = av_dict_copy(&out_fmt_ctx->metadata,
  in_fmt_ctx->metadata, AV_DICT_DONT_OVERWRITE)) < 0)
{
  fprintf(stderr, "Failed to copy file metadata.\n");
  goto end;
}

if ((ret =
  av_dict_set(&out_fmt_ctx->metadata, "title", title, 0)) < 0)
{
  fprintf(stderr,
    "Failed to set title for output format context.\n");
  goto end;
}
{{< /highlight >}}

First we will take in a third argument on the command line that will be used to
set the value of the metadata title for the output file. Next we will use the
`av_dict_copy` function to copy all metadata from the input file to the output
context. This data will be written to the file when we call
`avformat_write_header`. Finally, we will use `av_dict_set` to set the metadata
title on the output context to the value passed in on the command line.

## Chapters

{{< highlight c >}}
// in main
if ((ret = copy_chapters(out_fmt_ctx, in_fmt_ctx)) < 0)
{
  fprintf(stderr, "Failed to copy chapters.\n");
  goto end;
}
{{< /highlight >}}

{{< highlight c >}}
int copy_chapters(AVFormatContext *out_fmt_ctx,
  AVFormatContext *in_fmt_ctx)
{
  AVChapter *in_chapter, *out_chapter;
  int ret = 0;

  if (!(out_fmt_ctx->chapters =
    av_calloc(in_fmt_ctx->nb_chapters, sizeof(AVChapter *))))
  {
      fprintf(stderr,
        "Failed to allocate output format chapters array.\n");
      ret = AVERROR(ENOMEM);
      return ret;
  }

  for (int i = 0; i < in_fmt_ctx->nb_chapters; i++)
  {
    in_chapter = in_fmt_ctx->chapters[i];
    if (!(out_chapter = av_mallocz(sizeof(AVChapter)))) {
      fprintf(stderr,
        "Failed to allocate out_chapter for chapter %d", i);
      ret = AVERROR(ENOMEM);
      return ret;
    }

    out_chapter->id = in_chapter->id;
    out_chapter->time_base = in_chapter->time_base;
    out_chapter->start = in_chapter->start;
    out_chapter->end = in_chapter->end;

    if ((ret = av_dict_copy(&out_chapter->metadata,
      in_chapter->metadata, 0)) < 0)
    {
      fprintf(stderr, "Failed to copy chapter metadata.\n");
      av_freep(&out_chapter);
      return ret;
    }
    out_fmt_ctx->chapters[i] = out_chapter;
    out_fmt_ctx->nb_chapters++;
  }
  return ret;
}
{{< /highlight >}}

To copy chapters, we must first allocate the `chapters` array on the output
context. The size of the array will depend on the value of the `nb_chapters`
field on the input context. Then we'll loop over the chapters. For each one,
we'll get a reference to it from the `chapters` array field on the input
context. Then we'll allocate a new `AVChapter` struct and copy all data from
the input chapter to the new chapter. Then we'll use `av_dict_copy` to copy
any metadata from the input chapter to the new chapter. Finally, we'll asign
the new chapter to the `chapters` array field of the output context and
increment the `nb_chapters` field.

## Stream Metadata

{{< highlight c >}}
// in initialze_stream
if ((ret = av_dict_copy(&stream->metadata,
  in_fmt_ctx->streams[*stream_idx]->metadata,
  AV_DICT_DONT_OVERWRITE)) < 0)
{
  fprintf(stderr, "Failed to copy video metadata.\n");
  return ret;
}
{{< /highlight >}}

Considering how similar the code is for initializing the video stream and the
audio stream are, I broke this code out of the `main` function into it's own
`initialize_stream` function. The function takes in an `AVMediaType` value
which determines whether to initialize the audio stream or the video stream.
This value is given to `av_find_best_stream` and the stream index returned
is used to copy the codec parameters to the new stream just like before. In
addition, I've added the above code which again uses `av_dict_copy` to copy
the metadata from the input stream to the new stream.

With these new additions to the code, we will now produce files that have all
metadata and chapter information copied from the input file, so long as the
format container supports it. We can also set metadata values if we like. This
example only sets the file metadata `title` value but you can experiment to see
what other values you can set using `av_dict_set`.

In the next lesson we will learn how to add chapter markers to a file. We will
input a raw text file into `avformat_open_input` using the `ffmetadata` format.
The chapter data will parsed and the values will be set on `metadata` field of
the `AVFormatContext` struct which will then be copied to the output context
using `av_dict_copy`.

[Go To Previous Lesson - 1.1: Remux](/posts/libav-tutorial/lesson-1.1-remux/)

[View All Lessons](/tutorials/libav-tutorial/)
