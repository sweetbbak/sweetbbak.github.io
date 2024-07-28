+++
title = 'Greyscale Images Go'
author = 'sweetbbak'
date = 2024-07-14T10:38:43-08:00
draft = true
+++

Turning images into Greyscale in Golang...

<!--more-->

if you are looking for the simplest and fastest way to do this and want to just copy some code and gtfo of here, then here:

```go
func rgbaToGray(img image.Image) *image.Gray {
    var (
        bounds = img.Bounds()
        gray   = image.NewGray(bounds)
    )
    for x := 0; x < bounds.Max.X; x++ {
        for y := 0; y < bounds.Max.Y; y++ {
            var rgba = img.At(x, y)
            gray.Set(x, y, rgba)
        }
    }
    return gray
}
```

now, lets break down how an image is converted to grey scale.

here is a simple function that takes linearized RGB values as floats from the range 0-1

```go
func luminance64(r, g, b float64) float64 {
	return r*0.299 + g*0.587 + b*0.114
}
```
