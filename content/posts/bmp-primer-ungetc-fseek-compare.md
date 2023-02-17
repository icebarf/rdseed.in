+++ 
draft = false
date = 2023-02-17T09:07:45+05:30
title = "Comparing ungetc with fseek for going back in a stream, a few bits on BMP format"
description = "If you wish to set the file stream indicator back a few bytes, should use ungetc() or fseek() ?"
slug = ""
authors = ["Amritpal Singh"]
tags = ["C", "file streams", "bitmap", "images"]
categories = ["Programming", "Optimization", "File-Formats"]
externalLink = ""
aliases = ["bmp-primer-and-comparing-ungetc-with-fseek", "ungetc-vs-fseek", "bmp-primer", "bitmap-primer"]
series = []
+++

I've been working on a small hobby project/program of mine. It implements image rotation
by some angle (in degrees). I chose the bitmap format to write back the image that was
rotated, the input image is also expected to be a bitmap image.

Anyhow, while parsing I realised that the bitmap [file header](#bitmap-file-header) begins with a 2 byte
type number that identifies the BMP and DIB header. It is **commonly** `0x424d` (`BM` in ascii, `0x4d42` in little endian,
how it shall appear on my machine and probably yours as well.) 

Now, the goal is to read the first two bytes and compare those two bytes with `0x42` and `0x4d`
and then parse the header fully if we have the matching subset, else we don't parse.
During this, we need to return back to the start of the stream so we can collect the file header in one full `fread()`
call rather than manually setting our structure's type field[^1] and then `fread()` another few bytes
into our `header` structure[^2]

Here's an example of both cases (details and error checking have been omitted for brevity)

- `ungetc()`
```c
    short int byte1, byte2;
    byte1 = fgetc(bitmap);
    byte2 = fgetc(bitmap);

    if(byte1 != '0x42' && byte2 != '0x4d') { /* do not parse further */}

    /* what we care about */
    ungetc(byte2, bitmap);
    ungetc(byte1, bitmap);
```

- `fseek()` to stream start
```c
    short int byte1, byte2;
    byte1 = fgetc(bitmap);
    byte2 = fgetc(bitmap);

    if(byte1 != '0x42' && byte2 != '0x4d') { /* do not parse further */}

    /* what we care about */
    fseek(bitmap, 0, SEEK_SET);
```

Now, the question that arises, which is faster? Well the obvious thing is to benchmark this, so I
rolled up my little `get_time_nsec()`[^3] function and measured the time it took for both cases and calculated the delta.
I ran the benchmark 10 times and then took the average, the code was compiled with these compiler options.[^4]

Before I reveal the results, make a guess. Which is faster? I asked the same question to two of my friends and both agreed
that it should be `fseek()` which is faster. For the sake of keeping this article short, I shall not discuss the logic
behind their arguments.

## Results

- `ungetc()`
```py
ungetc_avg = (791+1793+1002+1012+1393+1002+1303+1583+1022+972)/10
ungetc_avg
1187.3
```

- `fseek()`
```py
fseek_avg = (3396+2936+3326+3356+1834+3576+2855+1783+3056+2285)/10
fseek_avg
2840.3
```

> The units for these numbers is `ns` or `nanosecond`.

This indicates that in this particular case, `ungetc()` is about 58.19% faster[^5].


## A better way to do things

### Bitmap File header
```c
struct __attribute((packed)) BitmapFileHeader
{
    u16 bf_type;
    u32 bf_fsize;
    u32 bf_reserved;
    u32 bf_pixels_offset;
} // 14-bytes
```
### Device-Independent Bitmap (BITMAPINFOHEADER)

```c
struct __attribute((packed)) BitmapInfoHeader_DIB
{
    u32 bi_size;
    i32 bi_width;
    i32 bi_height;
    u16 bi_panes;
    u16 bi_bitcount;
    u32 bi_compression;
    u32 bi_sizeimage;
    u32 bi_xpixels_permeter;
    u32 bi_ypixels_permeter;
    u32 bi_clrused;
    u32 bi_clrimportant;
} // 54-bytes
```

The previously described niche case is unrealistic and should not be used in real code.
Since we know that the bitmap file header i.e the the first 14-bytes are all the same, we could perform a read
of 14-bytes and fill out the file header, and then perform the check on `bf_type` and then parse the rest of DIB (Device-Independent Bitmap) header.

But there is another problem here, the DIB header has many versions with variying sizes, eg. 
`BITMAPCOREHEADER` (the oldest) is 12 bytes but `BITMAPINFOHEADER` (most common) is 54 bytes, and `OS22XBITMAPHEADER` adds
additional 24 bytes to the `BITMAPINFOHEADER`.

So we know the kind of mess we've got ourselves into. The subset that I have chosen to implement in my program is `BM` for file header type
and `BITMAPINFOHEADER` structure for the in-memory DIB. The goal is to only allow `BITMAPINFOHEADER` to be parsed (and optionally support others
in the future).

The initial 12-bytes of all DIB headers is the same (which is actually the `BITMAPCOREHEADER`), thus it is reasonable to perform another read
of 12-bytes and then conditionally parse the rest of the header based (similar to how it was done earlier[^2]) on `bi_size` field of the header.
This way we minimize fiddling with the stream and be done in about 3 or fewer reads.
Minimizing the number of reads is important so what we should ideally do is to have a packed struct with the file header and DIB header combined
and read 14 + 12 = 26 bytes in one read, perform the checks on `bf_type` and `bi_size` and report any errors if unsuccessful otherwise continue
parsing the rest of `BITMAPINFOHEADER`[^6]. Ofcourse, this advice is limited to the subset that was defined earlier. You may want to do things differently.

## Too Long didn't read

If you're parsing bitmap images and want to only support the most common sub-format in BMP format, then

- Perform a read of 26 bytes, check for file type and dib header size.[^6]
- parse further eg. the pixel array or the color profiles or the extra bit masks when there is specific case of compression of pixel data.
- Be done with it.

_and remember kids!_ Always minimize the number of reads or poking with the file stream.

## Footnotes

[^1]: `header->bf_type = (byte2 << 8) | byte1` or  if you read into a `short int` then simply `header->bf_type = id` perhaps

[^2]: Let us define an exemplary header

    ```c
    struct __attribute((packed)) BITMAPHEADER {
        u16 bf_type;
        u32 bf_fsize;
        u32 bf_reserved;
        u32 bf_pixels_offset;

        /* rest of the fields in the header */
        ...
    };
    ```

    Since we've manually set bf_type field already, we need to read into the structure from `bf_fsize` field.

    ```c
    fread (&header->bf_fsize, 1, sizeof(header) - sizeof(header->bf_type), bitmap);
    ```

[^3]: Benchmark 

    ```c
    long
    get_time_nsec (void)
    {
    #ifndef ICE_DONT_BENCHMARK_CODE
    struct timespec time;
    if (timespec_get (&time, TIME_UTC) == 0)
      fprintf (stdout,"Error: Filling timespec structure failed with timespec_get()\n");
    return time.tv_nsec;
    #endif
    }
    ```    

[^4]: `-Wall -Wextra -Werror -O3`

[^5]: ((fseek_avg - ungetc_avg) / fseek_avg) * 100 = 58.19807766785199% 

[^6]: Parsing the bitmap header in two reads. Error checking and some details have been omitted for the sake of brevity.

    ```c
    struct __attribute ((packed)) BITMAP_HEADER
    {
        u16 bf_id;
        u32 bf_fsize;
        u32 bf_reserved;
        u32 bf_pixels_offset;

        u32 bi_size;
        i32 bi_width;
        i32 bi_height;
        u16 bi_panes;
        u16 bi_bitcount;
        u32 bi_compression;
        u32 bi_sizeimage;
        u32 bi_xpixels_permeter;
        u32 bi_ypixels_permeter;
        u32 bi_clrused;
        u32 bi_clrimportant;
    }; // 54-bytes

    struct BITMAP_HEADER*
    read(FILE* bitmap)
    {
        struct BITMAP_HEADER *metadata = calloc(1, sizeof(*metadata));
        fread(data, 1, 26, bitmap);
        if(metadata->bf_id != 0x4d42) // little-endian machine, use switch if you wish to support multiple formats or something
            ...             // or perhaps do some macro magic, dealer's choice
        if(metadata->bi_size != 40)
            ...
        return metadata;
    }
    ```