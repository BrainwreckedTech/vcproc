vcproc - Video Clip Processor
=============================

Workflow
--------

           +-> remux ->+
           |     |     |
           |     ∨     |
    START -+   trim    ∨
           |     |     |          +-> preview ->-+-> youtube
           |     ∨     |          |              |
           +-->--+-->--+--> set ->+              +-> preview
                                  |              |
                                  +--> vcap -->--+-> join

Video Quality
-------------

The `youtube` process for `vcproc` uses x264's CRF 22.  While the idea was to hit Youtube's recommended bitrate of 12Mb/s, CRF has a wide range which depends on the compressability of the video.  CRF 22 has a range of -25% to +50% of the target 12Mb/s.

| CRF | Min bitrate | Max bitrate | Avg bitrate |
|-----|-------------|-------------|-------------|
|  19 |  13.10Mb/s  |  23.57Mb/s  |  18.97Mb/s  |
|  20 |  11.75Mb/s  |  21.50Mb/s  |  17.17Mb/s  |
|  21 |  10.53Mb/s  |  19.68Mb/s  |  15.54Mb/s  |
|  22 |   9.42Mb/s  |  18.10Mb/s  |  14.05Mb/s  |

The `portable` process for `vcproc` uses x264's CRF 29.  This process is intended for mobile devices with higher-density screens (which will lessen the impact of compression artifacts) and limited storage.

VCPROC Processes
----------------

### `all [process] [normal additional arguments]`

Execute `[process]` for each instance of vcproc.cfg (process=join|lossless|preview|vcap|youtube) or `[normal additional arguments]` (which would only be a file name when process=remux|set|trim) found in the current directory and all subdirectories.  If a `[process]` file (eg. vcap.mp4 for vcap, remux.ts for remux, etc.) is found, that directory will be skipped on the assumption that the process has already been run.

### `remux [video]`

This process might be optional, but there is no harm in running it, it operates almost as fast as your hard drive can copy the data, and it copies the original streams (maintaining their original quality) into the same container type with the time stamps reset to near-zero.

If your recorder generates "goofy" timestamps, this step will become necessary.  It will fix all time stamps so the remaining vcproc processes operate properly.  Contrary to popular belief, video doesn't have to start at time code 00:00:00.000.  It is perfectly valid for your first frame to start with a PTS (Presentation Time Stamp) that is something later -- sometimes **a lot** later.  In this event, the video recorder should also set the start_time for the stream appropriately.   Some video recorders don't and this will throw off `trim` and `vcap`.

### `trim [video]`

This process is optional but recommended to save space.  Chances are you don't want to save **all** of the video you recorded.  It operates much like REMUX in that the streams are still copied into the same container type they were found in, but `trim` will only copy the video between the In- and Out-Points you specify.

Actaully that's a bit of a lie, as `trim` will make two adjustments to your In-Point.  First, it will subtract two seconds because seeking is not accurate and this seems to be enough lead time to get the video you want.  (Oddly enough, during development, cutting on the I-frame directly before the In-Point resulted in video that always started at that point, which of course included up to one I-frame interval's worth of video that was not wanted.)  The second adjustment seeks backwards to the previous I-frame.

No adjustment is necessary for Out-Points, though it should be noted that `vcap` videos will continue to the closest I-frame after the Out-Point.  Processes that re-encode the video (PREVIEW, YOUTUBE, LOSSLESS, JOIN) will stick to the actual In-Point and Out-Point.

Additional note: No, you cannot simply copy the adjusted In- and Out-Points for use later.  I tried.  It failed.  The resulting time stamps will be slightly different.

### `set [video]`

This is where you define your clips properties.  The steps are now taken care of via dialog.  Behind the scenes, this is what is being written to the config file:

> #### VDOFIL

> Select the file you wish to operate on.  The file selection dialog can be skipped by providing a file name on the command line.

> #### OPHIGH, OPWIDE, FRMRAT, VDODAR (Output Height, Output Width, Framerate, Video Display Aspect Ratio)

> These values are auto-detected.  You can change these values if you want to convert your video to something different.

> #### VIP_TC, VOP_TC (Video In-Point Time Code, Video Out-Point Time Code)

> Set the In- and Out-Points for the video you want to save.

> #### MTLEAD, MTLAST (Mute Lead, Mute Last)

> Allows you to mute the first/last [x] seconds of audio.  Currently, MTLAST is non-functional.

> #### IPCRNG, OPCRNG (Input Color Range, Output Color Range)

> There are two standards for lumanince in video - pc/full/jpeg and tv/mpeg.  The pc/full/jpeg range is 0-255.  The tv/mpeg range is 16-235.  Optical media typically uses tv/mpeg.  Xbox 360 defaults to RGB (full), with the option to switch to YCbCr (mpeg).  PS3 always uses RGB (full) for games.  PS4 defaults to RGB Range: Limited with the option to change it to RGB Range: Full.  PC gaming, of course, uses pc/full/jpeg.  The tv/mpeg range will look slightly washed out on computer displays but fine on video displays.  The pc/full/jpeg range will crush shadows and lights on video displays but look fine on computer displays

> #### PVWIDE, PVHIGH (Preview Width, Preview Height)

> This allows you to set the dimensions for the optional PREVIEW step.

> #### ADPEAK

> This records the adjustment necessary to normalize audio to 0dB.

### `preview`

If you made no changes to your video's framerate or color range, do not use this step.  The `vcap` process is much quicker.  This step applies all video and audio settings and outputs a low-quality preview at [PVWIDE]x[PVHIGH] resolution.  Quality is of minimal concern at this point.  Getting a preview of your settings is.

### `vcap` (Video Copied, Audio Processed)

This step is intended for upload to YouTube, but is quick enough (due to the lack of video processing) that it can be used in place of PREVIEW.  If you don't need to change your video's dimensions, framerate, DAR, or color range, and your video was recorded at 18Mb/s or less, you might want to consider simply copying the video and only processing audio.  This way, no video quality loss will occur and you will get the audio filters you desire.

### `youtube`

This step is intended for upload to YouTube.  If your video was captured at a very high bitrate, or if you wanted to change the the video's dimensions, framerate, DAR, and/or color range, use this step.  All filters are applied and video is output at a CRF of 22.  Processing can take a while.

If your video was captured at a bit rate close to the target of 12Mb/s, and you aren't changing dimensions, framerate, DAR, or color range, try `vcap` instead.

### `join [video|vc] [video|vc] ...`

This step joins two or more videos.  The keyword "vc" is used to denote where you want the video clip defined by SET to appear.  Videos will be converted to (or created in) lossless FFV1 inside an MKV container using the filters you specified in SET.

Video will be created sequentially -- join0001.mkv, join0002.mkv, etc.  You can process clips ahead of time using LOSSLESS, link to them in the filesystem using the expected names, and bypass the time (and hard drive space) needed to use the same clip over and over.

### `lossless`

This process creates a lossless video.  It's purpose is to create videos for the JOIN process ahead of time.  Useful for clips that might be used over and over again, like intro, outros, and bumpers.  Simply SET these clips like you would any other video, then use filesystem links to refer back to them

### `thumb [seconds]`

Take a screenshot from the beginning (positive seconds) or end (negative seconds) of video.

This process has two behaviors:

> #### A config file from SET exists.

> The screenshot will be taken from the file defined in the config, using the In-Point and Out-Point defined therein as end points.

> #### No config file exists

> A screenshot will be taken of all videos within the current directory.

Unsupported Processes
---------------------

The following functions were created during development and have stagnated.

### `csav [video] [audio]` (Combine Seperate Audio and Video)

Combine the video from `[video]` with the audio from `[audio]`.

### homearch

This was meant as an alternative to creating lossless videos.  The only problem is that, for as much data as FFV1+FLAC can pump out, it is surprisingly quick at doing so.  A video that can be encoded in 45 minutes using x264+AAC in about 350MB can be encoded in 15 minutes using FFV1+FLAC in about 6-8 GB.
