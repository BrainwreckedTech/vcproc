vcproc - Video Clip Processor
=============================

Workflow
--------

           +-> remux
           |     |
           |     ∨
    START -+-> trim
           |     |
           |     ∨    +- preview -> youtube
           +->  set  -+
                      +-  vcap

### remux [video]

This process might be optional, but there is no harm in running it, it operates almost as fast as your hard drive can copy the data, and it copies the original streams (maintaining their original quality) into the same container type with the time stamps reset to near-zero.

If your recorder generates "goofy" timestamps, this step will become necessary.  It will fix all time stamps so the remaining vcproc processes operate properly.  Contrary to popular belief, video doesn't have to start at time code 00:00:00.000.  It is perfectly valid for your first frame to start with a PTS (Presentation Time Stamp) that is something later -- sometimes **a lot** later.  In this event, the video recorder should also set the start_time for the stream appropriately.   Some video recorders don't and this will throw off TRIM and VCAP.

### trim [video]

This process is optional but recommended to save space.  Chances are you don't want to save **all** of the video you recorded.  It operates much like REMUX in that the streams are still copied into the same container type they were found in, but TRIM will only copy the video between the In- and Out-Points you specify.

Actaully that's a bit of a lie, as TRIM will make two adjustments to your In-Point.  First, it will subtract two seconds because seeking is not accurate and this seems to be enough lead time to get the video you want.  (Oddly enough, during development, cutting on the I-frame directly before the In-Point resulted in video that always started at that point, which of course included up to one I-frame interval's worth of video that was not wanted.)  The second adjustment seeks backwards to the previous I-frame.

No adjustment is necessary for Out-Points, though it should be noted that VCAP videos will continue to the closest I-frame after the Out-Point.  Processes that re-encode the video (PREVIEW, YOUTUBE, LOSSLESS, JOIN) will stick to the actual In-Point and Out-Point.

Additional note: No, you cannot simply copy the adjusted In- and Out-Points for use later.  I tried.  It failed.  The resulting time stamps will be slightly different.

### set [video]

This is where you define your clips properties.

> #### VDOFIL

> Select the file you wish to operate on.  The file selection dialog can be skipped by providing a file name on the command line.

> #### OPHIGH, OPWIDE, FRMRAT, VDODAR (Output Height, Output Width, Framerate, Video Display Aspect Ratio)

> These values are auto-detected.  You can change these values if you want to convert your video to something different.

> #### VIP_TC, VOP_TC (Video In-Point Time Code, Video Out-Point Time Code)

> Set the In- and Out-Points for the video you want to save.

> #### MTLEAD, MTLAST (Mute Lead, Mute Last)

> Allows you to mute the first/last [x] seconds of audio.

> #### IPCRNG, OPCRNG (Input Color Range, Output Color Range)

> There are two standards for lumanince in video - pc/full/jpeg and tv/mpeg.  The pc/full/jpeg range is 0-255.  The tv/mpeg range is 16-235.  Optical media typically uses tv/mpeg.  Game consoles output to tv/mpeg unless you set them to use pc/full/jpeg.  PC gaming, of course, uses pc/full/jpeg.  The tv/mpeg range will look slightly washed out on computer displays but fine on video displays.  The pc/full/jpeg range will crush shadows and lights on video displays but look fine on computer displays 

> #### PVWIDE, PVHIGH (Preview Width, Preview Height)

> This allows you to set the dimensions for the optional PREVIEW step.

> #### ADPEAK

> This records the adjustment necessary to normalize audio to 0dB.

### preview

If you made no changes to your video's framerate or color range, do not use this step.  The VCAP process is much quicker.  This step applies all video and audio settings and outputs a low-quality preview at PVWIDExPVHIGH resolution.  Quality is of minimal concern at this point.  Getting a preview of your settings is.

### vcap (Video Copied Audio Processed)

This step is intended for upload to YouTube, but is quick enough (due to the lack of video processing) that it can be usued in place of PREVIEW.  If you don't need to change your video's dimensions, framerate, DAR, or color range, and your video was recorded at 18Mb/s or less, you might want to consider simply copying the video and only processing audio.  This way, no video quality loss will occur and you will get the audio filters you desire.

### youtube

This step is intended for upload to YouTube.  If your video was captured at a very high bitrate, or if you wanted to change the the video's dimensions, framerate, DAR, and/or color range, use this step.  All filters are applied and video is output at a CRF of 20, which should guarantee a bit rate of at least 12Mb/s (and use more if your video needs it).  Processing can take a while.

### join [video|vc] [video|vc] ...

This step joins two or more videos.  The keyword "vc" is used to denote where you want the video clip defined by SET to appear.  Videos will be converted to (or created in) lossless FFV1 inside an MKV container using the filters you specified in SET.

Video will be created sequentially -- join0001.mkv, join0002.mkv, etc.  You can process clips ahead of time using LOSSLESS, link to them in the filesystem using the expected names, and bypass the time (and hard drive space) needed to use the same clip over and over.

### lossless

This process creates a lossless video.  It's purpose is to create videos for the JOIN process ahead of time.  Useful for clips that might be used over and over again, like intro, outros, and bumpers.  Simply SET these clips like you would any other video, then use filesystem links to refer back to them

### thumb [seconds]

Take a screenshot from the beginning (positive seconds) or end (negative seconds) of video.

This process has two beviors:

> #### A config file from SET exists.

> The screenshot will be taken from the file defined in the config, using the In-Point and Out-Point defined therein as end points.

> #### No config file exists

> A screenshot will be taken of all videos within the current directory.

Unsupported Processes
---------------------

The following functions were created during development and have stagnated.

### csav [video] [audio] (Combine Seperate Audio and Video)

Combine the video from [video] with the audio from [audio].

### homearch

This was meant as an alternative to creating lossless videos.  The only problem is that, for as much data as FFV1+FLAC can pump out, it is surprisingly quick at doing so.  A video that can be encoded in 45 minutes using x264+AAC in about 350MB can be encoded in 15 minutes using FFV1+FLAC in about 6-8 GB.

