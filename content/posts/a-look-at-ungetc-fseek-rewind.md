+++ 
draft = false
date = 2023-02-17T09:07:45+05:30
title = "A look at ungetc, fseek and rewind."
description = "This article compares the efficiency and effectiveness of using ungetc, fseek, and rewind when seeking to go back to the start of a BMP image stream after reading some bytes. It also includes details about the BMP image structure."
slug = ""
authors = ["Icebarf"]
tags = []
categories = []
externalLink = ""
aliases = ["bmp-primer-and-comparing-ungetc-with-fseek", "ungetc-vs-fseek", "bmp-primer", "bitmap-primer"]
series = []
+++

## Index
- [The case of a few bytes](#the-case-of-a-few-bytes)
- [Results](#results)
- [Why is this happening?](#why-is-this-happening)
- [A better way to do things](#a-better-way-to-do-things)
- [Bitmap File header](#bitmap-file-header)
- [Device-Independent Bitmap](#device-independent-bitmap-bitmapinfoheader)
- [Too Long didn't read](#too-long-didnt-read)
- [Footnotes](#footnotes)

## The case of a few bytes
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

Here are some examples of the three cases. A few details and error checking have been omitted for brevity.

- `ungetc()`
```c
    short int byte1, byte2;
    byte1 = fgetc(bitmap);
    byte2 = fgetc(bitmap);

    if(byte1 != '0x42' && byte2 != '0x4d') { /* do not parse further */}

    /* what we care about */
    ungetc(byte2, bitmap);
    ungetc(byte1, bitmap); // probably check for eof because only 1 push back is guaranteed
```
Do note that with this approach only one push back is guaranteed as specified in the standard. Although, the
[cppreference's](https://en.cppreference.com/w/c/io/ungetc#Notes) notes section indicates that on popular operating systems
the pushback buffer is large enough to support our operations.

> The size of the pushback buffer varies in practice from 4k (Linux, MacOS) to as little as 4 (Solaris) or the guaranteed minimum 1 (HPUX, AIX).

> The apparent size of the pushback buffer may be larger if the character that is pushed back equals the character existing at that location in the external character sequence (the implementation may simply decrement the read file position indicator and avoid maintaining a pushback buffer). 

- `fseek()` to stream start
```c
    short int byte1, byte2;
    byte1 = fgetc(bitmap);
    byte2 = fgetc(bitmap);

    if(byte1 != '0x42' && byte2 != '0x4d') { /* do not parse further */}

    /* what we care about */
    fseek(bitmap, 0, SEEK_SET);
```

- `rewind()`
```c
    short int byte1, byte2;
    byte1 = fgetc(bitmap);
    byte2 = fgetc(bitmap);

    if(byte1 != '0x42' && byte2 != '0x4d') { /* do not parse further */}

    /* what we care about */
    rewind(bitmap)
```

Now, the question that arises, which is faster? Well the obvious thing is to benchmark this, so I
rolled up my little `get_time_nsec()`[^3] function and measured the time it took for both cases and calculated the delta.
I ran the benchmark 10 times and then took the average, the code was compiled with these compiler options.[^4]

Before I reveal the results, make a guess. Which is faster? I asked the same question to two of my friends and both agreed
that it should be `fseek()` which is faster (at the time, the choice for `rewind()` was not presented to them).
For the sake of keeping this article short, I shall not discuss the logic behind their arguments.

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

- `rewind()`
```py
rewind_avg = (4749+2374+5657+5727+4819+4889+6216+6146+5657+6984)/10
rewind_avg
5321.8
```

> The units for these numbers is `ns` or `nanosecond`.

This indicates that in this particular case, `ungetc()` is about 58.19% faster than `fseek()`[^5] and 77.68% faster than `rewind()`[^5].

From the presented options, it seems that going a little off-standard is worth this micro-optimization but I do not recommend it, as there
is better alternative to doing the above shenanigans. Although, this benchmark does have its issues, e.g includes time taken for
function calls but it does showcase a pretty large difference for a specific case.

## Why is this happening?

- `fseek()`

Let us take a look at glib source for `fseek()`. Upon inspection, it is defined as follows:

```c
int
fseek (FILE *fp, long int offset, int whence)
{
  int result;
  CHECK_FILE (fp, -1);
  _IO_acquire_lock (fp);
  result = _IO_fseek (fp, offset, whence);
  _IO_release_lock (fp);
  return result;
}
```

Looking for `_IO_fseek()` results in finding out that it is a macro that actually calls `_IO_seekoff_unlocked()` that further calls `_IO_seekoff()`.

```c
#define _IO_fseek(__fp, __offset, __whence) \
  (_IO_seekoff_unlocked (__fp, __offset, __whence, _IOS_INPUT|_IOS_OUTPUT) \
   == _IO_pos_BAD ? EOF : 0)
```

```c
off64_t
_IO_seekoff_unlocked (FILE *fp, off64_t offset, int dir, int mode)
{
  if (dir != _IO_seek_cur && dir != _IO_seek_set && dir != _IO_seek_end)
    {
      __set_errno (EINVAL);
      return EOF;
    }

  /* If we have a backup buffer, get rid of it, since the __seekoff
     callback may not know to do the right thing about it.
     This may be over-kill, but it'll do for now. TODO */
  if (mode != 0 && ((_IO_fwide (fp, 0) < 0 && _IO_have_backup (fp))
		    || (_IO_fwide (fp, 0) > 0 && _IO_have_wbackup (fp))))
    {
      if (dir == _IO_seek_cur && _IO_in_backup (fp))
	{
	  if (_IO_vtable_offset (fp) != 0 || fp->_mode <= 0)
	    offset -= fp->_IO_read_end - fp->_IO_read_ptr;
	  else
	    abort ();
	}
      if (_IO_fwide (fp, 0) < 0)
	_IO_free_backup_area (fp);
      else
	_IO_free_wbackup_area (fp);
    }

  return _IO_SEEKOFF (fp, offset, dir, mode);
}
```

Now, it seems that `fseek()` involves acquiring a look, calculating a bunch of stuff regarding the `backup buffer` it seems? And then
finally calling `_IO_SEEKOFF()`. This function seems to be invloving a lot of calls to other code in order to go back to the beginning
of the stream. This is a possible reason as to why it might be slow.

> Thanks to cortexauth#6190 for pointing me here.

- `rewind()`
After going through the source similarly at glibc, it is following the same pattern as `fseek()`.
```c
void
rewind (FILE *fp)
{
  CHECK_FILE (fp, );
  _IO_acquire_lock (fp);
  _IO_rewind (fp);
  _IO_clearerr (fp);
  _IO_release_lock (fp);
}
```
```c
#define _IO_rewind(FILE) \
  (void) _IO_seekoff_unlocked (FILE, 0, 0, _IOS_INPUT|_IOS_OUTPUT)
```

Where `rewind()` is also calling `_IO_seekoff_unlocked()` but with the offset and the stream position indicators being `0` and `0` respectively.
However, `rewind()` was actually slower than `fseek()` and that might be explained by what `_IO_clearerr()` might be doing which causes it to be
additionally slower than `fseek()`. But wait! `_IO_clearerr()` is actually a macro that modifies the `FILE` structure. It seems to be masking off
the error bits.

```c
#define _IO_clearerr(FP) ((FP)->_flags &= ~(_IO_ERR_SEEN|_IO_EOF_SEEN))
```

So what is the actual reason behind this slow down with `rewind()`? It could likely be the backup buffer mentioned in the source code, which is
being free'd. Take note that in the `ungetc()` example written above, there was a note from cppreference which indicated a pushback buffer
for the stream which might actually be coming into play with `_IO_seekoff_unlocked()`'s backup buffer here. It could likely be the same buffer(verification needed).
But then it should have been in effect when using `fseek()` as well. `rewind()` and `fseek()` should perform similar if not equally here in my opinion.

> Perhaps a more experienced reader could email me about this if they know more.

- `ungetc()`

Now, for our fastest contender `ungetc()` seems to be having a rather simple implementation. There is some minimal error checking and for
"putting back the character on the stream" it is just performing checks if the previous character in stream is the same character passed
as argument and then just decrements the `FILE` internal read counter/stream indicator.
```c
int
ungetc (int c, FILE *fp)
{
  int result;
  CHECK_FILE (fp, EOF);
  if (c == EOF)
    return EOF;
  if (!_IO_need_lock (fp))
    return _IO_sputbackc (fp, (unsigned char) c);
  _IO_acquire_lock (fp);
  result = _IO_sputbackc (fp, (unsigned char) c);
  _IO_release_lock (fp);
  return result;
}
```

```c
int
_IO_sputbackc (FILE *fp, int c)
{
  int result;

  if (fp->_IO_read_ptr > fp->_IO_read_base
      && (unsigned char)fp->_IO_read_ptr[-1] == (unsigned char)c)
    {
      fp->_IO_read_ptr--;
      result = (unsigned char) c;
    }
  else
    result = _IO_PBACKFAIL (fp, c);

  if (result != EOF)
    fp->_flags &= ~_IO_EOF_SEEN;

  return result;
}
```
> Above 8 code snippets (indicated by different code blocks) are directly taken from glibc source, their licensing applies these snippets respectively.

And there you have it! We finally have _some grasp_ over why `rewind()` and `fseek()` are slower than `ungetc()` in our specific case.

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

Considering [The case of a few bytes](#the-case-of-a-few-bytes) first, and then after the previous analysis,
the following conclusions can be drawn.

- `ungetc()` is about 58.19% faster than `fseek()`[^5] and 77.68% faster than `rewind()`[^5].
- `ungetc()` only modifies the internal structure of `FILE` and performs some checks.
- `fseek()` and `rewind()` are a lot more involved, calling many internal functions 
for acquiring locks, performing checks on pointers, handling a buffer etc.

If you're parsing bitmap images and want to only support the most common sub-format in BMP format, then

- Perform a read of 26 bytes, check for file type and dib header size.[^6]
- parse further eg. the pixel array or the color profiles or the extra bit masks when there is specific case of compression of pixel data.
- Be done with it.

_and remember,_ always minimize the number of reads or poking with the file stream.

Please email me if you wish to discuss these results with me, I would be delighted for any kind of feedback.

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

[^5]: `((original_value - new_value) / original_value) * 100 = X%`.
    Increaese/Decrease depends on whether the `original_value` was larger/smaller respectively.

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