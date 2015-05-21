rg\_etc1.cpp is a ZLIB licensed, performant, easy to use, and high quality 4x4 pixel block packer/unpacker for the ETC1 ([Ericsson Texture Compression](http://www.khronos.org/registry/gles/extensions/OES/OES_compressed_ETC1_RGB8_texture.txt)) texture compression format written in a single C++ source file. It has no external compile-time or run-time dependencies on any other library or API, allowing it to be trivially dropped in to other projects.

rg\_etc1 is **2-28x faster** than Ericsson's example ETC1 texture compressor used in the [etc-pack](http://devtools.ericsson.com/etc) tool, and is comparable in quality (RGB MSE). (As of this writing [5/13/12], and as far as I can determine, rg\_etc1.cpp/h is the fastest, and only known open source alternative to Ericsson's example compressor.)

The ETC/ETC1 texture compression format is supported by [OpenGL ES](http://en.wikipedia.org/wiki/OpenGL_ES) and is implemented in the GPU's used on many Android phones/tablets.


---

## How to Use ##

rg\_etc1.cpp/h only packs/unpacks individual ETC1 encoded 4x4 blocks, much like how [libsquish](http://code.google.com/p/libsquish/) or [ryg\_dxt](http://www.farbrausch.de/~fg/code/index.html) process DXT1/5 blocks. To process entire textures you'll need to call it separately on each block (ideally from multiple threads).

How to unpack 4x4 pixel blocks:

```
   // Unpacks an 8-byte ETC1 compressed block to an array of 4x4 32bpp RGBA pixels.
   // Returns false if the block is invalid. Invalid blocks will still be unpacked with clamping.
   // This function is thread safe, and does not dynamically allocate any memory.
   // If preserve_alpha is true, the alpha channel of the destination pixels will not 
   // be overwritten. Otherwise, alpha will be set to 255.
   bool unpack_etc1_block(const void *pETC1_block, unsigned int* pDst_pixels_rgba, bool preserve_alpha = false);
```

How to pack 4x4 pixel blocks:
```
   // Important: pack_etc1_block_init() must be called once sometime before calling pack_etc1_block().
   void pack_etc1_block_init();

   // Packs a 4x4 array of 32bpp RGBA pixels to an 8-byte ETC1 block.
   // 32-bit RGBA pixels must always be arranged as (R,G,B,A) (R first, A last) in 
   // memory, independent of platform endianness. A should always be 255.
   // Returns squared error of result.
   // This function is thread safe, and does not dynamically allocate any memory.
   // pack_etc1_block() does not currently support "perceptual" colorspace metrics. 
   // It primarily optimizes for RGB RMSE.
   unsigned int pack_etc1_block(void* pETC1_block, const unsigned int* pSrc_pixels_rgba, etc1_pack_params& pack_params);
```

See the header file [rg\_etc1.h](http://code.google.com/p/rg-etc1/source/browse/trunk/rg_etc1.h) for more details.


---

## Algorithm ##

rg\_etc1's compressor has three quality modes: low (but fast), medium (decent speed for single threaded compression on a Core i7 CPU), and high (intended for multithreaded compression). These quality modes are intended to be roughly comparable in quality to Ericsson's fast/medium/slow non-perceptual modes. rg\_etc1 uses several algorithms to efficiently compute the ETC1 block modes (flip/nonflipped, absolute/differential colors), the 444+444 or 555+333 subblock "base" colors, the intensity table to use for each subblock, and the sixteen (4x4) 2-bit selectors.

rg\_etc1 contains a simple, near-optimal solid block compressor that uses several small lookup tables. The solid color compressor is better than the results returned by Ericsson's example compressor in "slow" mode (measured using vanilla RGB MSE) approximately 9% of the time for random 24-bit input colors, and is equal ~90.9% of the time. See the function `pack_etc1_block_solid_color()` in [rg\_etc1.cpp](http://code.google.com/p/rg-etc1/source/browse/trunk/rg_etc1.cpp).

For non-solid color blocks, rg\_etc1 currently uses this algorithm: The outer loop iterates through each flip and differential possibility. There are 4 in all: non-flipped/differential colors, non-flipped/absolute colors, flipped/differential colors, and flipped/absolute colors. "Flipped" blocks use two 4x2 subblocks (top/bottom), and non-flipped use two 2x4 subblocks (left/right). The two subblock "base" colors can be coded using a differential scheme (555+333, where the 3-bit deltas range from [-4,3] in RGB 555 space), or non-differentially (444+444). Currently, independent of the quality mode, the outer loop tries each possibility to determine which flip/diff combination results in the least overall RMSE error over the entire 4x4 block.

For each 4x2 or 2x4 subblock, the compressor first computes the average subblock color, and (depending on the quality mode) it radix sorts the input subblock colors along the ETC1 intensity (1,1,1) axis. It then scans through a subset of the 3D lattice centered around the quantized, average subblock color trying each 3D (555 or 444) lattice point (discarding any forbidden lattice points while working on the second subblock in 555+333 differential color mode) as a potential solution. In low quality mode only a single lattice point is checked. In medium and high quality modes progressively larger lattice regions are scanned depending on the best error found so far.

There are two intensity table/pixel selector evaluation functions used to compute the error resulting from each trial quantized subblock color. The first, faster function (used in low and medium quality modes) enforces a [total ordering](http://en.wikipedia.org/wiki/Total_ordering) of pixel selectors along the ETC1 intensity axis (1,1,1). The second, used only in high quality mode, picks the best intensity table/selectors using plain RGB squared distance as an error metric.

Each time a better solution is found within the lattice scan loop, the compressor attempts to refine the current solution's quantized subblock color using a simple linear optimization based off the current selectors (and their resulting clamped RGB deltas), the current intensity table index, and the desired output colors. When successful, this optimization step produces a new subblock "base" color which results in less error. See the function, comments, and formulas in `etc1_optimizer::compute()` in [rg\_etc1.cpp](http://code.google.com/p/rg-etc1/source/browse/trunk/rg_etc1.cpp).

In high quality mode, the compressor also attempts to use the near-optimal solid color block compressor on solid-color 4x2 or 2x4 subblocks. (The small gain from this step may not justify the extra complexity/cost.)

In high quality mode, rg\_etc1's RGB PSNR is very close (typically within .3dB PSNR) of Ericsson's compressor in "slow" mode. I will create a wiki page with detailed statistics after I fully integrate rg\_etc1.cpp into [crunch](http://code.google.com/p/crunch).

rg\_etc1 optionally supports 555 [Floyd-Steinberg dithering](http://en.wikipedia.org/wiki/Floyd-Steinberg_dithering) of each 4x4 block prior to compression, using code borrowed from [ryg\_dxt's public domain DXTc block packer](http://www.farbrausch.de/~fg/code/index.html).


---

## Known Problems/Issues ##

rg\_etc1.cpp/.h is "fresh off the presses", and is the result of much experimenting and analysis. It has only been thoroughly tested when compiled with Visual Studio 2008 (x64 and x86). rg\_etc1 is actually a branch of the original version, which I'm currently integrating into the [crunch](http://code.google.com/p/crunch) texture compression project.

The algorithms used in rg\_etc1 where intended to efficiently scale to clusters containing hundreds or thousands of pixels, so some of the implementation choices are not optimal for tiny 4x4 pixel blocks.

"Fast" mode is definitely not as fast as it could be. The outer loop re-initializes the etc1\_optimizer class instance several times on the same subblocks, resulting in redundant sorting/average color/etc. computations. A radix sort is likely overkill for only 8 pixel subblocks. Even so, it still averages ~2.5x faster than Ericsson's compressor in "fast" mode.

rg\_etc1 does not yet support perceptual colorspace metrics. It always optimizes for RGB RMSE. (Except in "fast" and "medium" quality modes, which use a simpler, faster color matcher that tends to sacrifice RGB RMSE in favor of slightly higher Luma RMSE.)


---

## Alternatives/Links ##
  * [Mali Texture Compression Tool](http://www.malideveloper.com/developer-resources/tools/texture-compression-tool.php) - Closed source as far as I can tell, but it's supposed to be pretty fast.
  * [ETC Pack](http://devtools.ericsson.com/etc) - With source, but it uses a somewhat restrictive/complex license.
  * [ETC2 Presentation from Siggraph](http://www.khronos.org/assets/uploads/developers/library/2012-siggraph-opengl-es-bof/Ericsson-ETC2-SIGGRAPH_Aug12.pdf) - Given enough free time, I would like to implement support for ETC2


---

## Support Contact ##

For any questions or problems with this code please contact [Rich Geldreich](http://www.mobygames.com/developer/sheet/view/developerId,190072/) at <richgel99 at gmail.com>. Here's my [twitter page](http://twitter.com/#!/richgel999), and my [Google Plus](https://plus.google.com/u/0/106462556644344774154/posts) page.