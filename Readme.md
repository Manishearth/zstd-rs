# Ruzstd (a pure rust zstd decoder)

[![Released API docs](https://docs.rs/ruzstd/badge.svg)](https://docs.rs/ruzstd)
[![CI](https://github.com/killingspark/zstd-rs/workflows/CI/badge.svg)](https://github.com/killingspark/zstd-rs/actions?query=workflow%3ACI)


# What is this

A feature-complete decoder for the zstd compression format as defined in: [This document](https://github.com/facebook/zstd/blob/dev/doc/zstd_compression_format.md).

It is NOT a compressor. I don't plan on implementing that part either, at least not in the near future. (If someone is motivated enough I will of course accept a pull-request!)

This crate might look like it is not active, this is because there isn't really anything to do anymore, unless a bug is found or a new API feature is requested. I will of course respond to and look into issues!

# Current Status

This started as a toy project but it is in a usable state now.

In terms of speed it is still behind the original C implementation which has a rust binding located [here](https://github.com/gyscos/zstd-rs).

## Speed

Measuring with the 'time' utility the original zstd and my decoder both decoding the same enwik9.zst file from aramfs, my decoder is about 3.5 times slower. Enwik9 is highly compressible, for less compressible data (like a ubuntu installation .iso) my decoder comes close to only being 1.4 times slower.

## Can do:

1. Parse all files in /decodecorpus_files. These were generated with [decodecorpus](https://github.com/facebook/zstd/tree/dev/tests) by the original zstd developers
1. Decode all of them correctly into the output buffer
1. Decode all the decode_corpus files (1000+) I created locally
1. Calculate checksums
1. Act as a `zstd -c -d` dropin replacement
1. Can be compiled in a no-std environment that provides alloc

## Cannot do

This decoder is pretty much feature complete. If there are any wishes for new APIs or bug reports please file an issue, I will gladly take a look!

## Roadmap

1. More Performance optimizations (targets would be sequence_decoding and reverse_bitreader::get_bits. Those account for about 50% of the whole time used)
1. More tests (especially unit-tests for the bitreaders and other lower-level parts)
1. Find more bugs

## Testing

Tests take two forms.

1. Tests using well-formed files that have to decode correctly and are checked against their originals
1. Tests using malformed input that have been generated by the fuzzer. These don't have to decode (they are garbage) but they must not make the decoder panic

## Fuzzing

Fuzzing has been done with cargo fuzz. Each time it crashes the decoder I fixed the issue and added the offending input as a test. It's checked into the repo in the fuzz/artifacts/fuzz_target_1 directory. Those get tested in the fuzz_regressions.rs test.
At the time of writing the fuzzer was able to run for over 12 hours on the random input without finding new crashes. Obviously this doesn't mean there are no bugs but the common ones are probably fixed.

Fuzzing has been done on

1. Random input with no initial corpus
2. The \*.zst in /fuzz_decodecorpus

### You wanna help fuzz?

Use `cargo +nightly fuzz run decode` to run the fuzzer. It is seeded with files created with decodecorpus.

If (when) the fuzzer finds a crash it will be saved to the artifacts dir by the fuzzer. Run `cargo test artifacts` to run the artifacts tests.
This will tell you where the decoder panics exactly. If you are able to fix the issue please feel free to do a pull request. If not please still submit the offending input and I will see how to fix it myself.

# How can you use it?

Additionally to the descriptions and the docs you can have a look at the zstd / zstd_streaming binaries. They showcase how this library can be used.

## Easy

The easiest is to wrap the io::Read into a StreamingDecoder which itself implements io::Read. It will decode blocks as necessary to fulfill the read requests

```
let mut f = File::open(path).unwrap();
let mut decoder = StreamingDecoder::new(&mut f).unwrap();

let mut result = Vec::new();
decoder.read_to_end(&mut result).unwrap();
```

This might be a problem if you are accepting user provided data. Frames can be REALLY big when decoded. If this is the case you should either check how big the frame
actually is or use the memory efficient approach described below.

## Memory efficient

If memory is a concern you can decode frames partially. There are two ways to do this:

#### Streaming decoder

Use the StreamingDecoder and use a while loop to fill your buffer (see src/bin/zstd_stream.rs for an example). This is the
recommended approach.

#### Use the lower level FrameDecoder

For an example see the src/bin/zstd.rs file. Basically you can decode the frame until either a
given block count has been decoded or the decodebuffer has reached a certain size. Then you can collect no longer needed bytes from the buffer and do something with them, discard them and resume decoding the frame in a loop until the frame has been decoded completely.
