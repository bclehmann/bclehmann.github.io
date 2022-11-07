---
layout: post
title:  "SkiaSharp: Unmasking with SKColorFilter"
---

In my last [post]({% post_url 2022-11-05-HatchingWithSKShader %}), we created a shader that tiled a bitmap across the filled area. In this post, we're going to change that bitmap to be a black-and-white mask, and use a color filter to change the color on the fly.

Our `CreateBitmap` method doesn't change much, we simply hardcode the colours to black and white:

```cs
private static SKBitmap CreateBitmap()
{
  var bitmap = new SKBitmap(20, 50);

  using var paint = new SKPaint() { Color = SKColors.White };
  using var path = new SKPath();
  using var canvas = new SKCanvas(bitmap);

  canvas.Clear(SKColors.Black);
  canvas.DrawRect(new SKRect(0, 0, 20, 20), paint);

  return bitmap;
}

public readonly static SKBitmap Bitmap = CreateBitmap();
```

Since this function will always return the same thing, we cache the result, which, rather predictably, looks the same but with the colours swapped:

![Alternating black and white stripes going up and to the right](https://cdn-images-1.medium.com/max/2000/1*ly1s7o4LPxL0ncfq4NoqUw.png)

Now, how do we map white to our hatch colour, and black to our background colour? The key is [`SKShader.WithColorFilter`](https://learn.microsoft.com/en-us/dotnet/api/skiasharp.skshader.withcolorfilter?view=skiasharp-2.88), which will allow us to remap black and white to blue and red.

There are two main ways to create such an `SKColorFilter`. There's a function [`SKColorFilter.CreateTable`](https://learn.microsoft.com/en-us/dotnet/api/skiasharp.skcolorfilter.createtable?view=skiasharp-2.88#skiasharp-skcolorfilter-createtable(system-byte()-system-byte()-system-byte()-system-byte())) that takes four `byte[256]` arrays, which are interpreted as lookup tables for the A, R, G, and B channels respectively. For each colour c in the image, it applies this function:

```
c.A = alphaLookup[c.A]
c.R = redLookup[c.R]
c.G = greenLookup[c.G]
c.B = blueLookup[c.B]
```

Since our image only has two colours (black and white), you only need to set index 0 and index 255. For our purposes, this is a little wasteful, as it means allocating a kilobyte for what is a relatively simple filter. Unfortunately, be it out of concern for performance, or out of heretofore unknown masochism, I did not go with this option, I hope this was a good enough head start if you're interested.

The second way is to create a 4x5 matrix that all colours in our image will be multiplied by. I used to be a math minor, so I got a little overenthusiastic here. If you want to skip all the math you can jump to the bottom to see the code.

## The Math (copy-pastable code below)

From [this](https://learn.microsoft.com/en-us/xamarin/xamarin-forms/user-interface/graphics/skiasharp/effects/color-filters) article in the Xamarmin docs, the matrix looks like this:

$$
\begin{bmatrix}
 \text{M11} & \text{M12} & \text{M13} & \text{M14} & \text{M15} \\
 \text{M21} & \text{M22} & \text{M23} & \text{M24} & \text{M25} \\
 \text{M31} & \text{M32} & \text{M33} & \text{M34} & \text{M35} \\
 \text{M41} & \text{M42} & \text{M43} & \text{M44} & \text{M45} \\
\end{bmatrix}
\begin{bmatrix}
 r \\
 g \\
 b \\
 a \\
 1
\end{bmatrix} = 
\begin{bmatrix}
 r' \\
 g' \\
 b' \\
 a' \\
\end{bmatrix}
$$

The formulas for each channel are as follows:

$$
r' = \text{M11}·r + \text{M12}·g + \text{M13}·b + \text{M14}·a + \text{M15}
$$

$$
g' = \text{M21}·r + \text{M22}·g + \text{M23}·b + \text{M24}·a + \text{M25}
$$

$$
b' = \text{M31}·r + \text{M32}·g + \text{M33}·b + \text{M34}·a + \text{M35}
$$

$$
a' = \text{M41}·r + \text{M42}·g + \text{M43}·b + \text{M44}·a + \text{M45}
$$

This matrix can encode an arbitrary affine transformation (i.e. any linear transformation, with the addition of translation). That's why we have a five-dimensional colour, and it's why the matrix has five columns, so we can encode translations as well.

However, we only care what this matrix does to two colours, black and white, which are the vectors $[0, 0, 0, 0, 1]$ and $[1, 1, 1, 1, 1]$ respectively. So we're going to create a matrix that maps these vectors to our hatchColor and backgroundColor, which has the geometric interpretation of placing every shade of gray onto a line between hatchColor and backgroundColor.

We can start by translating all colours by the background colour $\textbf{bg}$. This maps black to $\textbf{bg}$, regardless of what the rest of the matrix looks like, as multiplying by the vector $[0, 0, 0, 0, 1]$ simply extracts the last column:

$$
\begin{bmatrix}
 \text{M11} & \text{M12} & \text{M13} & \text{M14} & bg_r \\
 \text{M21} & \text{M22} & \text{M23} & \text{M24} & bg_g \\
 \text{M31} & \text{M32} & \text{M33} & \text{M34} & bg_b \\
 \text{M41} & \text{M42} & \text{M43} & \text{M44} & bg_a \\
\end{bmatrix}
\begin{bmatrix}
 0 \\
 0 \\
 0 \\
 0 \\
 1
\end{bmatrix} = 
\begin{bmatrix}
 bg_r \\
 bg_g \\
 bg_b \\
 bg_a \\
\end{bmatrix}
$$

Now, we're going to define a vector $\textbf{diff} = \textbf{hatch}- \textbf{bg}$. This vector will form the main diagonal of our matrix, the rest of which will be zero:

$$
\begin{bmatrix}
 diff_r & 0 & 0 & 0 & bg_r \\
 0 & diff_g & 0 & 0 & bg_g \\
 0 & 0  & diff_b & 0 & bg_b \\
 0 & 0 & 0 & diff_a & bg_a \\
\end{bmatrix}
\begin{bmatrix}
 0 \\
 0 \\
 0 \\
 0 \\
 1
\end{bmatrix} = 
\begin{bmatrix}
 r' \\
 g' \\
 b' \\
 a' \\
\end{bmatrix}
$$

You may notice that, save for the last column, this is a diagonal matrix (aka a scaling matrix). Multiplying by a diagonal matrix is equivalent to multiplying the first component of the vector by the first element of the matrix's diagonal, the second component by second element of the diagonal, and so on. That is to say:

$$
c'_r = c_r \cdot diff_r
$$

$$
c'_g = c_g \cdot diff_g
$$

$$
c'_b = c_b \cdot diff_b
$$

$$
c'_a = c_a \cdot diff_a
$$

We can write this pairwise multiplication more concisely as $\textbf{c}' = \textbf{c} ⊙ \textbf{diff}$

Crucially, since the colour white is represented by the vector $[1, 1, 1, 1, 1]$, white maps to $\textbf{diff} + \textbf{bg}$. Since $\textbf{diff} = \textbf{hatch} - \textbf{bg}$, white maps to $\textbf{hatch}$. Now that we have a matrix that maps black to the background colour, and white to the hatch colour, we can start coding.

Note that in this explanation I made a simplifying assumption, namely that colour channels are on the interval $[0, 1]$ rather than $[0, 255]$. This is not true, so we have to divide our matrix by 255.

## SKColorFilter implementation

The code looks like this, note that I used foreground in place of hatchColor 

```cs
public static SKColorFilter GetMaskColorFilter(SKColor foreground, SKColor background)
{
  float redDifference = foreground.Red - background.Red;
  float greenDifference = foreground.Green - background.Green;
  float blueDifference = foreground.Blue - background.Blue;
  float alphaDifference = foreground.Alpha - background.Alpha;

  var mat = new float[] {
    redDifference / 255, 0, 0, 0, background.Red / 255.0f,
    0, greenDifference / 255, 0, 0, background.Green / 255.0f,
    0, 0, blueDifference / 255, 0, background.Blue / 255.0f,
    0, 0, 0, alphaDifference / 255, background.Alpha / 255.0f,
  };

  return SKColorFilter.CreateColorMatrix(mat);
}
```

Now, to actually use this colour filter, GetShader becomes the following:

```cs
public static SKShader GetShader(SKColor hatchColor, SKColor backgroundColor, StripeDirection stripeDirection = StripeDirection.Horizontal)
{
  var rotationMatrix = stripeDirection switch
  {
    StripeDirection.DiagonalUp => SKMatrix.CreateRotationDegrees(-45),
    StripeDirection.DiagonalDown => SKMatrix.CreateRotationDegrees(45),
    StripeDirection.Horizontal => SKMatrix.Identity,
    StripeDirection.Vertical => SKMatrix.CreateRotationDegrees(90),
    _ => throw new NotImplementedException(nameof(StripeDirection))
  };

  var shader = SKShader.CreateBitmap(
    Bitmap,
    SKShaderTileMode.Repeat,
    SKShaderTileMode.Repeat,
    SKMatrix.CreateScale(0.25f, 0.25f)
      .PostConcat(rotationMatrix));

  return shader.WithColorFilter(ColorFilterHelpers.GetMaskColorFilter(hatchColor, backgroundColor));
}
```

And this code gives the correct result for a hatch colour of red, and a background colour of blue:

![Alternating red and blue stripes going up and to the right](https://cdn-images-1.medium.com/max/2000/1*TPCU51q5_33FC9bVIAyB3g.png)

## Links

* Example code on github: [https://github.com/bclehmann/SkiaSharpHatching](https://github.com/bclehmann/SkiaSharpHatching)

* The pull request from which this code is based on: [https://github.com/ScottPlot/ScottPlot/pull/2221](https://github.com/ScottPlot/ScottPlot/pull/2221)
