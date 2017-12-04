# vcproc - Video Clip Processor

## Workflow

           +-> remux ->+
           |     |     |
           |     ∨     |
    START -+   trim    ∨
           |     |     |          +-> preview ->-+-> youtube
           |     ∨     |          |              |
           +-->--+-->--+--> set ->+              +-> preview
                                  |              |
                                  +--> vcap -->--+-> join

## Video Quality

The `youtube` and `join` processes for `vcproc` uses x264 with a default CRF of 22.  While the idea was to hit Youtube's recommended bitrate of 12Mb/s, CRF has a wide range which depends on how well the video can be compressed.  CRF 22 has a range of -25% to +50% of the target 12Mb/s.

| CRF | Min bitrate | Max bitrate | Avg bitrate |
|-----|-------------|-------------|-------------|
|  19 |  13.10Mb/s  |  23.57Mb/s  |  18.97Mb/s  |
|  20 |  11.75Mb/s  |  21.50Mb/s  |  17.17Mb/s  |
|  21 |  10.53Mb/s  |  19.68Mb/s  |  15.54Mb/s  |
|  22 |   9.42Mb/s  |  18.10Mb/s  |  14.05Mb/s  |

The `portable` process for `vcproc` uses x264's CRF 29.  This process is intended for mobile devices with higher-density screens (which will lessen the impact of compression artifacts) and limited storage.

| CRF | Min bitrate | Max bitrate | Avg bitrate |
|-----|-------------|-------------|-------------|
|  29 |   3.47Mb/s  |   9.15Mb/s  |   6.31Mb/s  |


## Environment Variables

At the moment, it does not matter what you set these environment variables to as `vcproc` only checks whether or not they are set.

### DISABLE_NND_COMPAT

If set, `vcproc` will skip `nnd_compat_check` which makes video PAR as close to 1:1 as possible.

### IGNORE_EXISTING

If set, `vcproc` will not skip processes if the the intended file already exists.

This is useful if you messed up one small detail and want to make sure you apply a single change across multiple jobs.

### USETMP

If set, `vcproc` will copy work files to `/tmp/vcproc-${USER}`, do its work there, and copy the finished results back to the source directory.

This is useful when the files reside on a network share.  If using multiple `vcproc` instances on multiple computers, this will avoid having every computer attempting to read from the file server all the time at the same time.

## VCPROC Processes

VCPROC has five sets of process: General, Utilities, Pre-Production, Production, and Unsupported.

### General

#### `all [processes] [dryrun] [normal additional arguments]`

Execute `[processes]` for each instance of `vcproc.cfg` (process=join|lossless|preview|vcap|youtube) or `[normal additional arguments]` (which would only be a file name when process=remux|set|trim) found in the current directory and all subdirectories.  If a `[process]` file (eg. `vcap.mp4` for `vcap`, `remux.ts` for `remux`, etc.) is found, that directory will be skipped on the assumption that the process has already been run.

The `[processes]` argument may be a single process or contain multiple processes separated by a plus sign.  Example: vcap+youtube+join+portable.

If `dryrun` is specified after `[process]` then `vcproc` will only print a list of the directories it will work on and then exit normally.

### Utilities

#### `fixar [video] [aspect]`

Sets the aspect ratio of a video to `[aspect]`.

#### `normalize [video]`

This process determines a video file's peak volume and increases that peak to 0dB.  Video is copied, not encoded.

#### `splitav [video]`

This process splits the video and audio into two separate files: `video.[ext]` and `audio.[ext]`.  For the video, `[ext]` will be the original video container's extension.  For the audio, vcproc will assign `[ext]` by parsing `ffprobe` output.

#### `thumb [seconds]`

Take a screenshot from the beginning (positive seconds) or end (negative seconds) of video.

This process has two behaviors:

> #### A config file from SET exists.
> The screenshot will be taken from the file defined in the config, using the In-Point and Out-Point defined therein as end points.

> #### No config file exists
> A screenshot will be taken of all videos within the current directory.

### Pre-production

#### `remux [video]`

This process might be optional, but there is no harm in running it, it operates almost as fast as your hard drive can copy the data, and it copies the original streams (maintaining their original quality) into the same container type with the time stamps reset to near-zero.

If your recorder generates "goofy" timestamps, this step will become necessary.  It will fix all time stamps so the remaining vcproc processes operate properly.  Contrary to popular belief, video doesn't have to start at time code 00:00:00.000.  It is perfectly valid for your first frame to start with a PTS (Presentation Time Stamp) that is something later -- sometimes **a lot** later.  In this event, the video recorder should also set the start_time for the stream appropriately.   Some video recorders don't and this will throw off `trim` and `vcap`.

#### `trim [video] [video] [video] ...`

This process is optional but recommended to save space.  Chances are you don't want to save **all** of the video you recorded.  It operates much like `remux` in that the streams are still copied into the same container type they were found in, but `trim` will only copy the video between the In- and Out-Points you specify.

Actually that's a bit of a lie, as `trim` will make some adjustments to your In-Point and Out-Points in favor of capturing extra video.

+ The In-Point will have three seconds subtracted from it
+ The In-Point will then be moved backwards to the closest I-frame (copied video must start on an I-Frame)
+ The Out-Point will have three seconds added to it.

The three-second slack space accounts for some "stickiness" with trimmed video.  The closer an In- or Out-Point is to the beginning or ending of a video, the more ffmpeg likes to "snap" that Point to an I-Frame.

You can specify multiple In- and Out-Points in one go.  This is great if you would like to split one large video into several smaller ones.

You can also perform the operation across multiple files.  Each file will be trimmed with the same In- and Out-Points, so make sure the files are aligned well enough to perform the operation.

#### `set [video]`

This is where you define your clips properties.  The steps are now taken care of via dialog.  Behind the scenes, this is what is being written to the config file:

> ##### VDOFIL
> Select the file you wish to operate on.  The file selection dialog can be skipped by providing a file name on the command line.

> ##### OPWIDE, OPHIGH, FRMRAT, VDODAR (Output Width, Output Height, Framerate, Video Display Aspect Ratio)
> These values are auto-detected.  You can change these values if you want to convert your video to something different.

> #####  CRPTOP, CRPLFT, CRPBTM, CRPRIT (Crop Top, Crop Left, Crop Bottom, Crop Right)
> Sets the number of pixels to crop from the top, left, bottom, and right.  Setting all four to zero (the default) disables cropping.  Video will scaled to OPWIDE x OPHIGH.

> ##### VIP_TC, VOP_TC (Video In-Point Time Code, Video Out-Point Time Code)
> Set the In- and Out-Points for the video you want to save.

> ##### MTLEAD, MTLAST (Mute Lead, Mute Last)
> Allows you to mute the first/last [x] seconds of audio.  Currently, MTLAST is non-functional.

> ##### IPCRNG, OPCRNG (Input Color Range, Output Color Range)
> There are two standards for lumanince in video - pc/full/jpeg and tv/mpeg.  The pc/full/jpeg range is 0-255.  The tv/mpeg range is 16-235.  Optical media typically uses tv/mpeg.  Xbox 360 defaults to RGB (full), with the option to switch to YCbCr (mpeg).  PS3 always uses RGB (full) for games.  PS4 defaults to RGB Range: Limited with the option to change it to RGB Range: Full.  PC gaming, of course, uses pc/full/jpeg.  The tv/mpeg range will look slightly washed out on computer displays but fine on video displays.  The pc/full/jpeg range will crush shadows and lights on video displays but look fine on computer displays.

> ##### PVWIDE, PVHIGH (Preview Width, Preview Height)
> This allows you to set the dimensions for the optional PREVIEW step.

> ##### ADPEAK
> This records the adjustment necessary to normalize audio to 0dB.

#### `preview [seconds]`

If you made no changes to your video's framerate or color range, do not use this step.  The `vcap` process is much quicker.

This step applies all video and audio settings and outputs a low-quality preview at `[PVWIDE]`x`[PVHIGH]` resolution.  Quality is of minimal concern at this point.  Getting a preview of your settings is.

If you don't want encode the entire clip (useful if you are checking audio sync) you can specify `[seconds]` to encode only a portion of the entire clip.

### Production

#### `vcap [mp4|mkv] [ifa|ifb]` (Video Copied, Audio Processed)

This step is intended for upload to YouTube, but is quick enough (due to the lack of video processing) that it can be used in place of `preview`.  If you don't need to change your video's dimensions, framerate, DAR, or color range, and your video was recorded at 18Mb/s or less, you might want to consider simply copying the video and only processing audio.  This way, no video quality loss will occur and you will get the audio filters you desire.

The default video container is `mp4` due to the intention of uploading to YouTube.  However, if you have other plans for the `vcap` file, you may find the `mkv` container is better-supported.

As video splicing without re-encoding requires starting on an I-Frame, your In-Point will not be strictly adhered to.  The default behavior starts the video with closest I-Frame *after* your In-Point (`ifa`).  You can choose to start the video with the closest I-Frame *before* the In-Point with the `ifb` argument.

#### `youtube [crf=nn|max=nn]`

This step is intended for upload to YouTube.  If your video was captured at a very high bitrate, or if you wanted to change the the video's dimensions, framerate, DAR, and/or color range, use this step.  If your video was captured at a bit rate close to the target of 12Mb/s, and you aren't changing dimensions, framerate, DAR, or color range, try `vcap` instead.  Processing can take a while, depending on the speed of your computer.

The default CRF value used is 22.  This can be changed with `crf=[nn]` (which will alter the CRF directly) or `max=[nn]` (which will alter CRF based on a desired maximum bitrate of `[nn]`Mb/s).

#### `join [crf=nn|max=nn] [video|vc] [video|vc] ...`

This step joins two or more videos.  The keyword "vc" is used to denote where you want the video clip defined by `set` to appear.  Videos will be converted to (or created in) lossless FFV1 inside an MKV container using the filters you specified in `set`.

Videos will be created sequentially -- `join0001.mkv`, `join0002.mkv`, etc.  You can process clips ahead of time using `lossless`, link to them in the filesystem using the expected names, and bypass the time (and hard drive space) needed to use the same clip over and over.

The default CRF value used is 22.  This can be changed with `crf=[nn]` (which will alter the CRF directly) or `max=[nn]` (which will alter CRF based on a desired maximum bitrate of `[nn]`MB/s).

#### `lossless`

This process creates a lossless video.  It's purpose is to create videos for the `join` process ahead of time.  Useful for clips that might be used over and over again, like intro, outros, and bumpers.  Simply `set` these clips like you would any other video, then use filesystem links to refer back to them.

### Unsupported Processes

The following functions were created during development and have stagnated.

#### `csav [video] [audio]` (Combine Seperate Audio and Video)

Combine the video from `[video]` with the audio from `[audio]`.
