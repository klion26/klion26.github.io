---
title: Arrow 中一次代码性能优化的记录
date: 2025-07-29 14:53:32
tags:
    - arrow
    - variant
    - parquet
    - rust
    - optimization
toc: true
---
> 本文记录 Arrow-RS 中一次性能优化的情况。

# 背景
最近尝试在 Arrow-rs 中做一些贡献来更好的学习了解下 Arrow 和 Rust。本文的事情则是在为 Arrow-rs 支持 parquet variant 中的一个修改。当前实现的过程中，Object 和 List 会创建临时 buffer，然后在最终 copy，这样会有一定的性能损失（主要在于临时 buffer 的创建以及拷贝），希望有一种方案能够直接写入到目标的 buffer 而不需要进行拷贝，也就是 [ARROW-RS-7977](https://github.com/apache/arrow-rs/issues/7977) 所记录的事情。

<!--more-->

# 过程
由于之前在 ARROW-RS#7899 中做过一个 Object 相关的，这个是类似的，因此觉得应该会是一个相对比较容易的事情。不过整个过程还是可以记录下

首先我给了一个最初的版本，这个版本中我写了四份代码，分别是

1. 使用 `PackedU32Iterator` 的[方案](https://github.com/apache/arrow-rs/pull/7987/commits/e4603c1d9b885c2f15871eddc87d1de45beb998e)
2. 纯粹使用 Iterator 的[方案](https://github.com/klion26/arrow-rs/blob/1d594b3b4a461a44ad72ecac730cbdfc537767d4/parquet-variant/src/builder.rs#L1220-L1240)
3. [该方案](https://github.com/klion26/arrow-rs/blob/7179a56258429d8431273d525ced836dd706e3e4/parquet-variant/src/builder.rs#L1220-L1241)在 2 的方式下，进行一次 iterator 的 collect，然后进行 splice
4. [该方案](https://github.com/klion26/arrow-rs/blob/9cc1b04a007c54274db81059d317747b2512e169/parquet-variant/src/builder.rs#L1265-L1285)直接生成一个临时的 buffer，然后逐步填充 header 后进行 splice

记录下 scovich 给的一些反馈
```
> 2. Splice with Iterator
This one will perform poorly because the chained iterator doesn't infer an accurate lower bound, so `Vec::splice` has to shift bytes twice (once to fit the lower bound, and again to fix the remainder).
> 3. Collect the header with iterator before splice
一定比 2 差，因为多一次 collect
> Splice with actual header bytes
This is still iterator-based like 1, but with all the unsafety of indexing into a pre-allocated temp buffer(and the overhead of allocating said temp buffer).
```
针对这几个方案，给出了两外两个建议

第 5 种方案
`A fifth approach would be to use the packed u32 iterator from 1/, and splice in a pre-populated temp buffer like 5/, but to populate the temp buffer by push+extend calls instead of chain+collect:`

```rust
let mut bytes_to_splice = vec![header];
  ...
bytes_to_splice.extend(num_elements_bytes);
  ...
bytes_to_splice.extend(offsets);
  ...
bytes_to_splice.extend(data_size_bytes);
buffer
    .inner_mut()
    .splice(starting_offset..starting_offset, bytes_to_splice);
```

以及第六种，主要是优化了原来的 `PackedU32Iterator`
`A sixth approach would also use a pre-populated temp buffer, but ditch the packed u32 iterator from 1/ and just directly append the bytes:`

```rust
fn append_packed_u32(dest: &mut Vec<u8>, value: u32, value_bytes: usize) {
    let n = dest.len() + value_bytes;
    dest.extend(value.to_le_bytes());
    dest.truncate(n);
}

// Calculated header size becomes a hint; being wrong only risks extra allocations.
// Make sure to reserve enough capacity to handle the extra bytes we'll truncate.
let mut bytes_to_splice = Vec::with_capacity(header_size + 3);
bytes_to_splice.push(header);

append_packed_u32(&mut bytes_to_splice, num_elements, if is_large { 4 } else { 1 });

for offset in std::mem::take(self.offsets) {
    append_packed_u32(&mut bytes_to_splice, offset as u32, offset_size as usize);
}

append_packed_u32(&mut bytes_to_splice, data_size as u32, offset_size as usize);

buffer
    .inner_mut()
    .splice(starting_offset..starting_offset, bytes_to_splice);
```

然后进行了 benchmark 的测试，发现第六种在所有的方案中会更好



这里有一些性能的点
- `Vec::splice` 需要 `Iterator` 的 lower-bound 来进行预测，不然可能会耗时更多，可以通过自定义的 Iterator 来提供
- 使用 extend 这些可能会性能更好（？）
- 可以通过对 U32 进行处理，直接完成 PackedU32 相关的事情，其中 U32 可以当成是 4 个 U8 然后 `to_le_bytes()` 就是按照顺序来排列的，这样的话，可以使用 `truncate`来轻量级的处理只需要 U32 部分 byte 的情况
- 另外在 Rust 中 `into_iter()` 和 `iter()` 这些还需要继续学习了解下。


整体看 Rust 中是可以获得更好的性能，这些需要对相对更低层的知识有更好的了解（或者这里更多的是对 API 有更好的了解），这些如果仅仅通过学习是比较难了解到的，通过实际的项目则可以更好/直观的看到不同实现的差异。
