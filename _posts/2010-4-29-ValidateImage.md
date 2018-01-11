---
layout: post
title: Validate Image In C#
---

A simple way to check if a given stream has an image header.

When we work with files we use stream. The problem is that you expect a stream of an image/picture but actually you get something else - probably a stream of a renamed 500GB iso file. So it is a good practice to validate and be sure that the file is actually an image/picture before reading the whole file. This is a simple task because each image/picture type has a header with unique value at the beginning if the file. So we will get the first few bytes from our file and compare them with statically defined headers for each image/picture type.

```c#
//  Defined image headers.
private static byte[][] imageHeaders = new byte[][]
{
    new byte[]{ 0xFF, 0xD8 },                                       //  .jpg, .jpeg, .jpe, .jfif, .jif
    new byte[]{ 0x42, 0x4D},                                        //  .bmp
    new byte[]{ 0x89, 0x50, 0x4E, 0x47, 0x0D, 0x0A, 0x1A, 0x0A },   //  .png
    new byte[]{ 0x47, 0x49, 0x46 }                                  //  .gif
};

/// <summary>
/// Validate that the stream is of an image file.
/// </summary>
/// <remarks>
/// IMPORTANT: The calling code is responsible for creating and disposing the image stream.
/// Supported file types: .JPEG .BMP .GIF .PNG
/// </remarks>
/// <param name="imageStream">The stream of a picture file</param>
/// <exception cref="Exception">Throws if the stream is of invalid image</exception>
/// <returns></returns>
public static bool IsValidImage(Stream imageStream)
{
    if(imageStream.Length > 0)
    {
        byte[] header = new byte[8]; // Change size if needed.
        imageStream.Read(header, 0, header.Length);

        bool hasImageHeader = imageHeaders.Count(magic =>
        {
            int i = 0;
            if(magic.Length > header.Length)
                return false;
            return magic.Count(b => { return b == header[ i++ ]; }) == magic.Length;
        }) > 0;

        return hasImageHeader;
    }

    return false;
}
```

In [next post][1] we will apply the same technique to validate video streams

------------------------------

Software is fun! Happy coding!

------------------------------

[1]: https://google.com