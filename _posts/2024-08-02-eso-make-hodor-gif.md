---
layout: post
title: "Making animated avatar for Hodor Reflexes in ESO"
author: "Shtille"
categories: journal
tags: [ESO,bash,python]
image: avatar/smb_cropped.png
---

Once I've decided to make an animated avatar for ESO's Hodor Reflexes Add-on.

## Requirements

The avatar requirements are:
- size 32x32
- no more than 50 frames
- format .gif or .dds (Direct Draw Surface)

## The process

### Source selection

I concluded that the most fitting avatar and the most I've wanted is one from computer game "Super Meat Boy".

### Video recording

I've recorded a video with Geforce Experience tool.

<img src="{{ '/assets/img/avatar/smb_source.png' | relative_url }}">

### Cropping video

The captured video has format 1920x1080, so I cropped it with *ffmpeg* into region 80x80:

```bash
ffmpeg -i source.mp4 -vf "crop=80:80:925:692" cropped.mp4
```

The crop region was taken from GIMP by editing one frame image.

<img src="{{ '/assets/img/avatar/smb_cropped.png' | relative_url }}">

### Split video into frames

Next step is splitting video into frames.

```bash
ffmpeg -i cropped.mp4 frames/img%04d.png
```

The provided video has been split into 115 frames.

### Determine unique frames

Then we need to determine unique frames. No wonder that many frames just coincide with siblings.
So unique frame numbers are:

```py
numbers=[1,4,8,12,16,22,26,30,34,38,42,48,52,56,60,64,68,74,78,82,86,90,94,100,104,108,112]
```

As we can see it's not so many unique frames.

### Determine key frames

There are three key frames for this animation:

1) Left leg forward

<img src="{{ '/assets/img/avatar/smb_key_left.png' | relative_url }}">

2) Stand still

<img src="{{ '/assets/img/avatar/smb_key_stand.png' | relative_url }}">

3) Right leg forward

<img src="{{ '/assets/img/avatar/smb_key_right.png' | relative_url }}">

So let's find out frame numbers with equal positions:

- left: 8,60,112
- stand: 16,42,68,94
- right: 34,86

As we can see, full animation period is 52 frames.

### Isolating unique frames

Next we gonna isolate unique cropped frames with numbers from 8 to 60 into separate folder.

```py
numbers=[8,12,16,22,26,30,34,38,42,48,52,56]
```

So there are 12 unique frames. Store in separate directory:

```py
import shutil

numbers = [8,12,16,22,26,30,34,38,42,48,52,56]
src_format = "./crop/frames/img{number:04d}.png"
dst_format = "./unique/img{number:02d}.png"

i = 0
for x in numbers:
	src = src_format.format(number = x)
	dst = dst_format.format(number = i)
	shutil.copy2(src, dst)
	i += 1
```

### Remove background

We gonna remove background (make it transparent) for each frame.
At first copy unique frames to separate directory.

```py
import shutil

src_format = "./unique/img{number:02d}.png"
dst_format = "./transparent/img{number:02d}.png"

for i in range(12):
	src = src_format.format(number = i)
	dst = dst_format.format(number = i)
	shutil.copy2(src, dst)
```

Then remove background for each frame in GIMP editor.

<img src="{{ '/assets/img/avatar/smb_transparent.png' | relative_url }}">

### Consolidate frames

Next we gonna fill missing as they were before unique frames isolation.
Note that stand key frames have 6 frames gap to the next unique frame instead of regular 4 frames.

```py
import shutil

numbers = [8,12,16,22,26,30,34,38,42,48,52,56]
stand = [16,42]
stand_indices = []

def get_stand_indices():
	for s in stand:
		stand_indices.append(numbers.index(s))

get_stand_indices()

# Determines if this index corresponds to stand key frame
def is_stand(index):
	for i in stand_indices:
		if index == i:
			return True
	return False

src_format = "./transparent/img{number:02d}.png"
dst_format = "./solid/img{number:02d}.png"

# We fill from 8 to 60
index = 0
for i in range(len(numbers)):
	src = src_format.format(number = i)
	count = 4
	if is_stand(i):
		count = 6
	for j in range(count):
		dst = dst_format.format(number = index)
		index += 1
		shutil.copy2(src, dst)
# End loop
src = src_format.format(number = 0)
dst = dst_format.format(number = 52) # = 60 - 8
shutil.copy2(src, dst)
```

### Make GIF from images

At first make original size GIF:

```bash
src="./solid/img%02d.png"
dst="avatar80.gif"
palette="/tmp/palette80.png"
filters="fps=30"
framerate=60

ffmpeg -framerate $framerate -i $src -vf "$filters,palettegen" -y $palette
ffmpeg -framerate $framerate -i $src -i $palette -lavfi "$filters [x]; [x][1:v] paletteuse" -loop 0 -y $dst
```

The result will be:

<img src="{{ '/assets/img/avatar/smb_avatar80.gif' | relative_url }}">

Then make GIF 32x32 as required:

```bash
src="./solid/img%02d.png"
dst="avatar32.gif"
palette="/tmp/palette32.png"
filters="fps=30,scale=32:-1:flags=lanczos"
framerate=60

ffmpeg -framerate $framerate -i $src -vf "$filters,palettegen" -y $palette
ffmpeg -framerate $framerate -i $src -i $palette -lavfi "$filters [x]; [x][1:v] paletteuse" -loop 0 -y $dst
```

The result will be:

<img src="{{ '/assets/img/avatar/smb_avatar32.gif' | relative_url }}">

### Convert GIF to DDS

The easiest solution would be using ImageMagick:

```bash
magick -format dds -define dds:compression=none avatar32.gif avatar32.dds
```

But the result is just one image, not the series of sprites. And here comes the internet solution: [ezgif](https://ezgif.com/gif-to-sprite).

## Consclusion

So we made a GIF with 27 frames:

<img src="{{ '/assets/img/avatar/smb_avatar32.gif' | relative_url }}">