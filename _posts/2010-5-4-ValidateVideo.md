---
layout: post
title: Validate Video In C#
---

A simple way to check if a given stream has a video header.

In my previous post I showed you [how to validate an image/picture file][1]. Today I will add a video support to this validation and I will refactor the code a bit. Full source code is bellow:

```c#
public enum FileType
{
    Picture,
    Audio,
    Video,
    Text,
    Other,
}

public static class FileValidator
{
    private static Dictionary<FileType, byte[][]> _definedHeaders = new Dictionary<FileType, byte[][]>( 2 );

    //  Defined video headers.
    private static byte[][] _videoHeaders = new byte[][]
    {
        new byte[]{ 0x41, 0x56, 0x49, 0x20 }, // .avi
        new byte[]{ 0x52, 0x49, 0x46, 0x46 }, // .avi
        new byte[]{ 0x30, 0x26, 0xB2, 0x75, 0x8E, 0x66, 0xCF, 0x11, 0xA6, 0xD9, 0x00, 0xAA, 0x00, 0x62 }, // .asf, .wmv
        new byte[]{ 0x00, 0x00, 0x01, 0xB3 }, // .mpg
        new byte[]{ 0x00, 0x00, 0x01, 0xba}, // .mpg
        new byte[]{ 0x00, 0x00, 0x00, 0x18}, // .mp4
        new byte[]{ 0x46, 0x4C, 0x56 } // .flv
    };

    //  Defined image headers.
    private static byte[][] _imageHeaders = new byte[][]
    {
        new byte[]{ 0xFF, 0xD8 },                                       //  .jpg, .jpeg, .jpe, .jfif, .jif
        new byte[]{ 0x42, 0x4D},                                        //  .bmp
        new byte[]{ 0x89, 0x50, 0x4E, 0x47, 0x0D, 0x0A, 0x1A, 0x0A },   //  .png
        new byte[]{ 0x47, 0x49, 0x46 }                                  //  .gif
    };

    static FileValidator()
    {
        RegisterHeaders();
    }

    private static void RegisterHeaders()
    {
        _definedHeaders.Add( FileType.Picture, _imageHeaders );
        _definedHeaders.Add( FileType.Video, _videoHeaders );
    }

    /// <summary>
    /// Validate that the stream is of an expected type.
    /// </summary>
    /// <remarks>
    /// IMPORTANT: The calling code is responsible for creating and disposing the stream.
    /// </remarks>
    /// <param name="theStream">The stream of a file.</param>
    /// <exception cref="Exception">Throws if the stream is of invalid file.</exception>
    /// <returns></returns>
    public static bool IsValidFile( Stream theStream, FileType ofTFileype )
    {
        if( theStream.Length > 0 )
        {
            byte[] header = new byte[ 8 ]; // Change size if needed.
            theStream.Read( header, 0, header.Length );

            bool hasHeader = _definedHeaders[ ofTFileype ].Count( magic =>
            {
                int i = 0;
                if( magic.Length > header.Length )
                    return false;
                return magic.Count( b => { return b == header[ i++ ]; } ) == magic.Length;
            } ) > 0;

            return hasHeader;
        }

        return false;
    }

    public static bool IsValidFile( FileInfo fileInfo, FileType ofFileType )
    {
        bool result = fileInfo.Exists;

        if( IsFileTypeRegistered( ofFileType ) )
        {
            using( Stream stream = fileInfo.OpenRead() )
            {
                result = result ? FileValidator.IsValidFile( stream, ofFileType ) : false;

                if( !result )
                    throw new Exception( string.Format( "Invalid {0} file. {1}", ofFileType.NameToString(), fileInfo.FullName ) );
            }
        }

        return result;
    }

    private static bool IsFileTypeRegistered( FileType fileType )
    {
        return _definedHeaders.ContainsKey( fileType );
    }

    public static bool IsValidFile( string file, FileType ofFileType )
    {
        return IsValidFile( new FileInfo( file ), ofFileType );
    }
}
```

------------------------------

Software is fun! Happy coding!

------------------------------

[1]: https://google.com