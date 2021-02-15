---
layout: post
title: "Making a GIF from a Video"
author: "Antti Petaisto"
tags: Tooling, Reference
---

There's a lot of different ways to convert videos or images into a GIF, here's a way that I found to work for my usecase, creating GIF's to show steps of software applications.  As with so much of what I do, I stand on the shoulder of giants who create the open source software that I use.

## Get the Software

I use Ubuntu, so the steps I show for installing software are for it. Steps will be different for other OS's. Once the requisite software the remaining steps should be the same.  
```
sudo apt upgrade
sudo apt install ffmpeg 
sudo apt install imagemagick
```
You'll also need Python, installation instructions for Python are [here](https://www.python.org/downloads/).

## Video to Image Frames

We first need to convert our video into images. We'll do this by using ffmpeg.

```
ffmpeg -ss 00:00 -i "$VIDEOLOCATION" -t "$VIDEOEND" "$IMAGEFILEDIRS"/%04d.png
```

The %04d.png will start at a file called `0001.png` and increment up from there. 

## Grab a Subset of Image Frames

Depending on the frame rate we may have a lot of images. My video had a framerate of 30 frames per second and for a gif we don't want that frequent of updates.  I created a Python script to only grab the first 2 and the last 2 images as well as every 20th one.  This reduces the number of images making for a much smaller files size.

``` Python
import os
import shutil
import sys

# get input
image_location = sys.argv[1]
# determine directory location and set gif_file location
directory_location = image_location[0:image_location.rindex('/')]
gif_location = directory_location + '/gif_files/'
# create gif_location dir
if os.path.isdir(gif_location):
    shutil.rmtree(gif_location)
os.mkdir(gif_location)
print('image_location: ' + image_location)
directory = os.fsencode(image_location)
#count number of png image files in dir
num_pngs = len([name for name in os.listdir(image_location) if os.path.isfile(image_location + '/' + name) and name.endswith('.png') ])
print('num_pngs: ' + str(num_pngs))
    
# iterate through directory and grab first 2 and last 2 pngs as well as every 20th and copy to gif_files directory
for file in os.listdir(directory):
    filename = os.fsdecode(file)
    if filename.endswith(".png"): 
        filename_number = filename[:-4]
        if int(filename_number) % 20 == 0 or int(filename_number) in (1, 2, num_pngs, num_pngs-1):
           shutil.copy(image_location+ '/' + filename, gif_location + filename)
```

At some future date I plan to develop a more "intelligent" file_grabber that detects changes as well.

## Convert to GIF
Once we have a subset of all the frame images we can convert it to a GIF. We'll use ImageMagick to do this. 

This converts the files in the gif_files directory to a gif with a 75ms delay.

```
convert -delay 75 -loop 0 "$VIDEODIR"/gif_files/*.png new_gif.gif
```

## Putting all steps together

I used a bash script to put all the steps together. This allows me to run the script with a video as an input and output a GIF that's much smaller than the raw output from ffmpeg

``` bash
#!bin/bash

# inputs
VIDEOLOCATION=$1
VIDEOSTART=$2
VIDEOEND=$3

# getting dir
VIDEODIR=$(dirname "$VIDEOLOCATION")
echo $VIDEODIR
IMAGEFILEDIRS="$VIDEODIR"/image_files
GIFFILEDIRS="$VIDEODIR"/gif_files

# removing existing directorys and recreating
rm -rf "$IMAGEFILEDIRS"
mkdir "$IMAGEFILEDIRS"
echo $IMAGEFILEDIRS
rm -rf "$GIFFILEDIRS"
mkdir "$GIFFILEDIRS"
sudo chmod a+wrx "$IMAGEFILEDIRS"

# converting video to images
ffmpeg -ss 00:00 -i "$VIDEOLOCATION" -t "$VIDEOEND" "$IMAGEFILEDIRS"/%04d.png

# grabbing a subset of images
python3 /home/persamina/file_grabber.py $IMAGEFILEDIRS

# converting images into gifs
convert -delay 75 -loop 0 "$VIDEODIR"/gif_files/*.png new_gif.gif
```

