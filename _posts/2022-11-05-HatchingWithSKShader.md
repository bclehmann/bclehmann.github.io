
# SkiaSharp: Hatched fills with SKShader

Coming from System.Drawing.Common, one of the things I missed most about SkiaSharp was the lack of support for hatched fills out of the box. If you're unfamiliar, hatching allows you to paint with a pattern applied:

![Source: [https://scottplot.net/cookbook/4.1/category/plottable-bar-graph/#bar-fill-pattern](https://scottplot.net/cookbook/4.1/category/plottable-bar-graph/#bar-fill-pattern)](https://cdn-images-1.medium.com/max/2000/0*t16rJbhPb6xLa5vZ.png)

*Source: [https://scottplot.net/cookbook/4.1/category/plottable-bar-graph/#bar-fill-pattern](https://scottplot.net/cookbook/4.1/category/plottable-bar-graph/#bar-fill-pattern)*

In SkiaSharp, you have two options:

* [SKPathEffect](https://learn.microsoft.com/en-us/dotnet/api/skiasharp.skpatheffect?view=skiasharp-2.88)

* [SKShader](https://learn.microsoft.com/en-us/dotnet/api/skiasharp.skshader?view=skiasharp-2.88)

If you want very simple results, SKPathEffect may be appropriate, but I would not recommend it for interactive or real-time rendering. SKShader isn't much harder to implement, and SKPathEffect has significantly poorer performance, especially if the hatched area takes up large portions of the screen. The same zoom level could yield ~10 FPS with SKPathEffect, and ~300 FPS with shaders.

Additionally, SKPathEffect has some further drawbacks. If you draw a circle with SKPathEffect, the effect will bleed over significantly. And SKPathEffect will not tile up to the edges of the circle, yielding poor-looking results. These can be countered by clipping and shrinking the tiling unit respectively, but for my purposes it wasn't worth it. Especially as reducing the size of each tile further exacerbates the performance problems. The rest of this post will be completely focused on SKShader.

## SKShader

So, you've decided to use SKShader? Despite the name, it's really not too hard to use, you don't have to (and to my knowledge, cannot) write shaders from scratch in SkiaSharp. Instead, we're going to create a small bitmap, and use [SKShader.CreateBitmap](https://learn.microsoft.com/en-us/dotnet/api/skiasharp.skshader.createbitmap?view=skiasharp-2.88#skiasharp-skshader-createbitmap(skiasharp-skbitmap-skiasharp-skshadertilemode-skiasharp-skshadertilemode)) to create a shader that tiles it across the fill.

For a striped hatch, the code to create the bitmap looks like this:

```cs
public static SKBitmap CreateBitmap(SKColor hatchColor, SKColor backgroundColor)
{
    var bitmap = new SKBitmap(20, 50);

    using var paint = new SKPaint() { Color = hatchColor };
    using var path = new SKPath();
    using var canvas = new SKCanvas(bitmap);

    canvas.Clear(backgroundColor);
    canvas.DrawRect(new SKRect(0, 0, 20, 20), paint);

    return bitmap;
}
```

And the bitmap itself (for a hatchColor of red, and a backgroundColor of blue):

![](https://cdn-images-1.medium.com/max/2000/1*1Gcz9mKCsd5KAuupZZ9npw.png)

Doesn't look like much, does it? In any case, it's enough to create a striped pattern. Now we have to create a shader:

```cs
public static SKShader GetShader(SKColor hatchColor, SKColor backgroundColor)
{
    return SKShader.CreateBitmap(
       CreateBitmap(hatchColor, backgroundColor),
       SKShaderTileMode.Repeat,
       SKShaderTileMode.Repeat,
       SKMatrix.CreateScale(0.25f, 0.25f));
}
```

Now, if we use this shader to paint a square:

```cs
var shader = GetShader(SKColors.Red, SKColors.Blue);
using var bmp = new SKBitmap(128, 128);
using var canvas = new SKCanvas(bmp);
using var paint = new SKPaint()
{
    Shader = shader
};

canvas.DrawRect(new(0, 0, 128, 128), paint);
WriteBitmapToFile(bmp, "hatch.png"); // This function is included in the github link at the bottom
```

![](https://cdn-images-1.medium.com/max/2000/1*B5vXoO3Yrsd62C616Y8opw.png)

Note that the 2nd and 3rd parameters set the tiling mode for the x and y directions respectively. In our case, we want it to repeat, but if you want it to mirror or clamp in one direction you can. Clamping in this context means displaying the image once and stretching the last pixel to the edge of the fill area.

The last parameter is for applying a transformation to the shader. For now, we just want to rescale it, but soon we'll use it for rotations.

Now, what if you want vertical or diagonal stripes? You could rotate the bitmap, but it's simpler to rotate the shader. That way, you can reuse the same bitmap for different rotations.

Since the last parameter to SKShader.CreateBitmap was a transformation matrix, we can simply multiply by our desired rotation matrix to get what we want:

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

    return SKShader.CreateBitmap(
        CreateBitmap(hatchColor, backgroundColor),
        SKShaderTileMode.Repeat,
        SKShaderTileMode.Repeat,
        SKMatrix.CreateScale(0.25f, 0.25f)
            .PostConcat(rotationMatrix));
}
```

Now, if we call GetShader with StripeDirection.DiagonalUp, we get this result instead:

![](https://cdn-images-1.medium.com/max/2000/1*TPCU51q5_33FC9bVIAyB3g.png)

A quick note on the transformation matrix, when you call A.PostConcat(B) it represents this matrix multiplication A * B. These matrices represent a linear transformation, but matrix multiplication is not commutative. Unintuitively, the multiplication A * B corresponds to B(A(x)).

So while our code might look like it scales the shader followed by a rotation, it actually rotates and then scales. In this case, it doesn't make a difference, but it's worth noting in case you get up to anything more complicated.

For most people's purposes, this is as far as they need to go. But if you're astute, you may have noticed that we baked the colours (in our case red and blue) into the bitmap. What if we need to be able to provide the same pattern with multiple colour palettes, do we have to regenerate the bitmap each time? Or do we have to write extra code to invalidate the bitmap should the colours change?

Instead, we can store the bitmaps as black and white, and then add the colour later. Then we can cache these bitmaps without ever needing to invalidate them. Seeing as they're so small, you could even bundle them with your distribution, should you be so inclined.

Our CreateBitmap method doesn't change much, we simply hardcode the colours to black and white:

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

Since this function will always return the same thing, we cache the result, which, rather predictably, looks like this:

![](https://cdn-images-1.medium.com/max/2000/1*ly1s7o4LPxL0ncfq4NoqUw.png)

Now, how do we map white to our hatch colour, and black to our background colour? The key is [SKShader.WithColorFilter](https://learn.microsoft.com/en-us/dotnet/api/skiasharp.skshader.withcolorfilter?view=skiasharp-2.88),  which will allow us to remap black and white to blue and red.

There are two main ways to create such an SKColorFilter. There's a function [SKColorFilter.CreateTable](https://learn.microsoft.com/en-us/dotnet/api/skiasharp.skcolorfilter.createtable?view=skiasharp-2.88#skiasharp-skcolorfilter-createtable(system-byte()-system-byte()-system-byte()-system-byte())) that takes four   byte[256] arrays, which are interpreted as lookup tables for the A, R, G, and B channels respectively. For each colour c in the image, it applies this function:

    c.A = alphaLookup[c.A]
    c.R = redLookup[c.R]
    c.G = greenLookup[c.G]
    c.B = blueLookup[c.B]

Since our image only has two colours (black and white), you only need to set index 0 and index 255. For our purposes, this is a little wasteful, as it means allocating a kilobyte for what is a relatively simple filter.

Unfortunately, be it out of concern for performance, or out of heretofore unknown masochism, I did not go with this option, I hope this was a good enough head start if you're interested.

The second way is to create a 4x5 matrix that all colours in our image will be multiplied by. I used to be a math minor, so I got a little overenthusiastic here. If you want to skip all the math you can jump to the bottom to see the code.

From [this](https://learn.microsoft.com/en-us/xamarin/xamarin-forms/user-interface/graphics/skiasharp/effects/color-filters) article in the Xamarmin docs, the matrix looks like this:

![](https://cdn-images-1.medium.com/max/2000/1*ilysm8PNMH2ls81_cGWobw.png)

The formulas for each channel are as follows:

![](https://cdn-images-1.medium.com/max/2186/1*mwfZOz_A0RK1Y_KYNxXq5Q.png)

This matrix can encode an arbitrary affine transformation (i.e. any linear transformation, with the addition of translation). That's why we have a five-dimensional colour, and it's why the matrix has five columns, so we can encode translations as well.

However, we only care what this matrix does to two colours, black and white, which are the vectors [0, 0, 0, 0, 1] and [1, 1, 1, 1, 1] respectively. So we're going to create a matrix that maps these vectors to our hatchColor and backgroundColor, which has the geometric interpretation of placing every shade of gray onto a line between hatchColor and backgroundColor.

We can start by translating all colours by backgroundColor. This maps black to the backgroundColor, regardless of what the rest of the matrix looks like, as multiplying by the vector [0, 0, 0, 0, 1] simply extracts the last column:

![](https://cdn-images-1.medium.com/max/2000/1*LeV2bO-O74AldFs1DxaX_A.png)

Now, we're going to define a vector, diff = hatchColor  ΓÇö  backgroundColor. This vector will form the main diagonal of our matrix, the rest of which will be zero:

![](https://cdn-images-1.medium.com/max/2000/1*QdFQOFn-tkFrj25IDfi18w.png)

You may notice that, save for the last column, this is a diagonal matrix (aka a scaling matrix). Multiplying by a diagonal matrix is equivalent to multiplying the first component of the vector by the first element of the matrix's diagonal, the second component by second element of the diagonal, and so on.

This means, that for a colour c, multiplying by this matrix is equivalent to the following:

    c.R = c.R * diff.R + bg.R
    c.G = c.G * diff.G + bg.G
    c.B = c.B * diff.B + bg.B
    c.A = c.A * diff.A + bg.A

Crucially, since the colour white is represented by the vector [1, 1, 1, 1, 1], white maps to diff + backgroundColor. Since diff = hatchColor - backgroundColor, white maps to hatchColor.

In this explanation I made a simplifying assumption, that colour channels are on the interval [0, 1] rather than [0, 255]. This is not true, so we have to divide our matrix by 255.

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

    var filter = SKColorFilter.CreateColorMatrix(mat);
    return filter;
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

And this code gives the correct result:

![](https://cdn-images-1.medium.com/max/2000/1*TPCU51q5_33FC9bVIAyB3g.png)

* Example code on github: [https://github.com/bclehmann/SkiaSharpHatching](https://github.com/bclehmann/SkiaSharpHatching)

* The pull request from which this code is based on: [https://github.com/ScottPlot/ScottPlot/pull/2221](https://github.com/ScottPlot/ScottPlot/pull/2221)
