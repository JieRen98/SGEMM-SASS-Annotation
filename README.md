# SGEMM-SASS-Annotation

## Background

The project [MaxAs](https://github.com/NervanaSystems/maxas) is a famous assembler project for NVIDIA Maxwell architecture. It gives a well-known sass demo SGEMM for the matrix multiplication which has a better performance than cuBLAS. However, not only the tutorial ("SGEMM.pdf" in this repo) but also the sass code ("sgemm64_origin.sass" in this repo) are oversimplified. I strongly recommend Chinese readers refer to the [blog](https://www.jianshu.com/p/e01024892afb) by XiaoyuWang for a detailed theoretical explanation. As far as I knew, there is no annotated sass code when I write this markdown file.

## How-to

The annotated code is in "sgemm64.sass", take some codes as example:

```assembly
--:-:1:-:1      S2R R119, SR_TID.X;                                 # R119 = threadIdx.x, tid
--:-:2:-:1      S2R R125, SR_CTAID.X;                               # R125 = blockIdx.x, bx
--:-:3:-:1      S2R R122, SR_CTAID.Y;                               # R122 = blockIdx.y, by
01:-:-:-:1      ISETP.GE.AND P0, PT, R119.reuse, 0x20, PT;          # P0 = tid >= 32
--:-:-:-:1      LOP.AND R9, R119.reuse, 0xf;                        # R9 = tid & 0xf, tid15
```

Words after `#` are my annotation, annotation `R119 = threadIdx.x, tid` means the 119th register stores the value of `threadIdx.x` and it has a alias `tid`; annotation `R9 = tid & 0xf, tid15` means the 9th register stores the result of `tid & 0xf` and the alias of the 9th register is `tid15`. Often, the alias is corresponding to the alias in the pseudo-code given by "SGEMM.pdf":

```c
tid = threadId.x;
bx = blockId.x;
by = blockId.y;
blk = tid >= 32 ? by : bx;
ldx = tid >= 32 ? ldb/4 : lda/4;
tex = tid >= 32 ? texB : texA;
```

Here is another example:

```assembly
--:-:-:-:1      ISCADD R112, R8, R9, 0x4;                           # R112 = blk << 4 + tid15, _track0

...

--:-:-:-:1      XMAD.MRG R5, R1.reuse, R4.H1.reuse, RZ;             # R5 = ldx * tid2, _track0

...

--:-:-:Y:6      XMAD R112, R1.reuse, R4, R112;                      # R112 = ldx * tid2 + R112, track0
```

According to the tutorial, variable `track0` is calculated by the following operation:

```c
track0 = blk*64/4 + tid15 + (ldx * tid2);
```

However, it is impossible to calculate `track0` in one step. The intermediate variables produced by the intermediate process are all named with an underline in front of the final alias: `_track0`.

In the example:

```assembly
--:-:-:-:1      LDS.U.128 R64, [R114];                              # R64 ~ R67 = [readAs+4*(0*64+0) ~ readAs+4*(0*64+0)+3*4]
--:-:-:-:1      LDS.U.128 R72, [R115];                              # R72 ~ R75 = [readBs+4*(0*64+0) ~ readBs+4*(0*64+0)+3*4]
--:-:-:-:1      LDS.U.128 R68, [R114+0x80];                         # R68 ~ R71 = [readAs+4*(0*64+32) ~ readAs+4*(0*64+32)+3*4]
--:-:1:-:1      LDS.U.128 R76, [R115+0x80];                         # R76 ~ R79 = [readBs+4*(01*64+32) ~ readBs+4*(0*64+32)+3*4]
```

Because the instruction `LDS.U.128` will load 4 4-Bytes numbers, there will be 4 registers with the continuous index will be loaded from the shared memory.

## About

The code is given by Scott Gray (scott[at]openai.com) in the project MaxAs, annotation is given by Jie Ren (jieren9806[at]gmail.com). Because of my limited knowledge, there would be some mistakes in my annotation, feel free to discuss with me. I will be appreciated for your valuable comments.
This project is released under the [MIT License](https://github.com/JieRen98/SGEMM-SASS-Annotation/blob/main/LICENSE).
