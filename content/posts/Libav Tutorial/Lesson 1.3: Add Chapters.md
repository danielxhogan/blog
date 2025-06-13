+++
title = 'Lesson 1.3: Add Chapters'
date = 2025-06-06T11:58:08-07:00
draft = false
tutorials = 'Libav Tutorial'
+++

![christine lens flare](/images/LibavTutorial/Lesson_1.3/christine_lens_flare.jpg)

In this lesson, you will learn how add chapter markers to a video file by
storing chapter information in a text file and inputting the file into
`avformat_open_input`. As with the previous lesson, there are test videos in the
`videos/inputs` folder and test commands in the `run.sh` file. Running these
will inject the chapter information from `chapters.txt` into the input file
and produce a new file with chapter markers, depending on whether the output
format supports chapters. All the code for this tutorial can be found
[here](https://github.com/danielxhogan/LibavTutorial/tree/main/Lesson%201%3A%20Remux/Lesson%201.3%3A%20Add%20Chapters).

## Chapter File Structure

{{< highlight c >}}
;FFMETADATA1
[CHAPTER]
TIMEBASE=1/1000
START=0
END=6000
title=Introduction
hi=bye

[CHAPTER]
TIMEBASE=1/1000
START=5000
END=10000
title=Main Content

[CHAPTER]
TIMEBASE=1/1000
START=10000
END=12815
title=Conclusion
{{< /highlight >}}

The first line in the `chapters.txt` file is `;FFMETADATA1`. This will help
`avformat_open_input` auto detect the format when opening the file. Next, each
chapter marker starts with the `[CHAPTER]` indicator. Next, we must specify the
timebase. Here we are specifying that the timestamps will be in milliseconds.
If this value is not provided, the default value is `1/1,000,000,000`. Next, we
must specify a `START` time and an `END` time. The end time must come after the
start time for the same chapter, however, the end time can come after the start
time of the next chapter. For `mp4`, `mov`, and `wmv`, the end time listed in
the files metadata will be the start time of the next timestamp, regardless of
the value stated in the `chapters.txt` file, unless it is the last chapter. For
`mkv` and `webm`, the end time listed in the files metadata will be the value
stated in the `chapters.txt` file, but the actual chapter marker when playing
the file will be at the start of the latter chapter, not at the end of the
former. `mkv` also supports arbitrary metadata values such as `hi=bye`. These
values will be igonred by the other formats.

## Opening Chapter File

{{< highlight c >}}
const char *v_filename, *ch_filename, *out_filename, *title;

AVFormatContext *v_fmt_ctx = NULL, *ch_fmt_ctx = NULL,
  *out_fmt_ctx = NULL;

const AVInputFormat *ch_fmt;

...

ch_filename = argv[2];

...

if (!(ch_fmt = av_find_input_format("ffmetadata"))) {
  fprintf(stderr, "Failed to find format for chapter file.\n");
  ret = AVERROR_UNKNOWN;
  goto end;
}

if ((ret = avformat_open_input(&ch_fmt_ctx,
  ch_filename, ch_fmt, NULL)) < 0)
{
  fprintf(stderr, "Failed to open chapter file: '%s'.\n",
    ch_filename);
  goto end;
}
{{< /highlight >}}

The first major difference in this example is that we are taking in two inputs
instead of one. We must declare an `AVFormatContext` struct that will hold
information about the `chapters.txt` file and an `AVInputFormat` struct that will
hold information about the format of the file. We assign the second
command line argument as the name of the chapters file.

We then call `av_find_input_format` specifying the `ffmetadata` format and
assign the return value to `ch_fmt`. Technically, in this case it is not
necessary to specify the format becuase the file begins with `;FFMETADATA1` but
if it didn't, `avformat_open_input` would still be able to parse it. I just
wanted to show two different ways of indicating what type of file this is.

Now we can call `avformat_open_input` passing in the filename and format.
Information about the file will be populated in `ch_fmt_ctx` including a
`chapters` array just like the `chapters` array from the last example on the
video file that already had chapter information.


## Adding Chapters

{{< highlight c >}}
if ((ret = copy_chapters(out_fmt_ctx, ch_fmt_ctx)) < 0)
{
  fprintf(stderr, "Failed to copy chapters.\n");
  goto end;
}
{{< /highlight >}}

The next major difference is that when we call `copy_chapters`, we pass in the
`AVFormatContext` for the `chapters.txt` file instead of the video file. The
`copy_chapters` function is identical to the fuction in the previous example.
At this point, we have now injected chapter information into `out_fmt_ctx`,
only the information came from a text file and not an input video file. From
here on, the rest of the files is identical to the code from the previous example.

In the next lesson, we will learn how to inject subtitles into a file. The
process will be fairly similar to this one. We will have two inputs, the input
video file and another text file with subtitle information. The file will be
parsed into an `AVFormatContext` struct that will have a subtitle stream on it
just like if we had input a video file that had a subtitle stream on it. This
stream will be muxed into the output file.

[Go To Previous Lesson - 1.2: Copy Metadata](/posts/libav-tutorial/lesson-1.2-copy-metadata/)

[View All Lessons](/tutorials/libav-tutorial/)