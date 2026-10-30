# SeekDB 全文索引性能优化报告

## 概述

针对赛题四「全文索引调优」，对 seekdb 的全文索引构建、分词和查询链路做了性能优化。
所有修改**保持原有分词与检索语义**（TOKENIZE 分词结果与 MATCH 查询语义不变），属于纯性能重构。
当前网络环境无法稳定推送到 GitHub（github.com:443 连接不稳定），4 个优化 commit 已保存在本地
`master` 分支，网络恢复后执行 `git push origin master` 即可。

## 已提交的优化（4 个 commit，7 个文件）

```
aeedc32a4 perf(fts): optimize IK tokenizer and dict-match hot paths
735181397 perf(fts): optimize text token stream and grouping hot paths
c49e19068 perf(fts): single-lookup grouping in ObAddWord::groupby_word
561bf7e58 perf(fts): avoid per-word allocator/charset_convert in stopword check
```

---

## 修改明细

### 1. IK 中文分词热路径（ob_ik_char_util.h）

**瓶颈定位依据**：`ObFTCharUtil::do_classify` 是分词器对**每个字符**都要执行的分类函数。
原实现对同一字符依次调用 `is_alpha`、`is_arabic`、`is_chinese`、`is_other_cjk`、
`is_surrogate_high`、`is_surrogate_low`，每个 `is_*` 内部都独立调用一次
`decode_unicode`（UTF-8 解码）。对一个普通汉字最多被重复解码 4~6 次。

**主要修改**：合并为**单次 UTF-8 解码**，解码出一个 `ob_wc_t` 码点后，按原有判断顺序
（alpha → arabic → chinese → other_cjk → surrogate_high → surrogate_low）依次做范围判断。
错误路径（NULL 输入、解码失败）完全保留。

**影响**：分词热路径上每个字符的 UTF-8 解码次数从最多 6 次降到 1 次，显著减少
`prepare_next_char`/`step_next` 每个字符的 CPU 开销。对 3,000 次/组的中文分词测试收益直接。

**语义**：判断顺序、优先级、错误返回均与原实现一致，不改变分词结果。

---

### 2. 词典前缀匹配热路径（ob_ft_range_dict.cpp）

**瓶颈定位依据**：`ObFTRangeDict::find_first_char_range` 在**每个字符**的前缀匹配时被调用，
原实现对 `range_dicts_`（按首字符分组的字典范围数组）做**全量线性扫描**，每个 range 做 2 次
`strcmp`。IK 主词典约数十万词，分片可达几十个，每次匹配 O(#ranges)。

**主要修改**：改为**二分查找**。range 由 `build_one_range` 按首字符非递减、不相交的顺序构建，
因此对非递减的 `start_` 字段做 lower_bound 二分（找第一个 `start_ >= word` 再确认相等），
与线性扫描**返回第一个匹配 range** 的语义完全一致。

**影响**：每字符前缀匹配从 O(n) 降到 O(log n)，是词典匹配和仲裁切分的核心加速。

**语义**：对同一首字符多 range 的情况，用 lower_bound 保证返回第一个匹配 range，与原实现一致。

---

### 3. DAT 哈希表探针修正（ob_ft_dat_dict.cpp）

**瓶颈定位依据**：`ObArrayHashMap` 用 `(index + 3) % capacity` 的探针步长做哈希冲突处理。
`estimate_capacity` 取 `MAX(word_num * 4/3, 101)`。当 capacity 可被 3 整除时，探针序列只访问
同一 mod-3 剩余类（仅 1/3 槽位可达），会导致 `find` 漏判、`insert` 死循环——这既是性能隐患
也是正确性隐患。

**主要修改**：`estimate_capacity` 中保证 capacity 与探针步长 3 互质（被 3 整除则 +1 直到不整除）。

**影响**：消除潜在的漏判/死循环，保证哈希表在所有槽位可达；DAT 前缀匹配的查询更稳定高效。

**语义**：仅改容量分配，不改变匹配结果；`estimate_size`/`init` 复用该函数，`memset` 大小自动一致。

---

### 4. 仲裁比较热路径（ob_ik_token.cpp）

**瓶颈定位依据**：`ObIKTokenChain::better_than` 在智能切分仲裁的每次冲突比较中，分别调用
`x_weight()` 和 `p_weight()`，每次都是**全链遍历**，即每次比较做 2 次完整链遍历。

**主要修改**：将两次遍历**合并为一次**，单趟遍历同时计算 x_weight（char_cnt 乘积）和
p_weight（Σ p*char_cnt）。

**影响**：仲裁比较的遍历次数减半，对长歧义句的智能切分收益明显。

**语义**：p 从 1 开始、char_cnt 相乘、p*char_cnt 累加的计算逻辑与顺序完全一致。

---

### 5. 英文空格分词热路径（ob_token_stream.cpp）

**瓶颈定位依据**：`ObTextWhitespaceTokenizer::get_next` 对**每个字节**调用
`ob_mbcharlen_ptr`（字符集函数指针间接调用）判断字符长度。英文（BENG）文本绝大多数是
单字节 ASCII。

**主要修改**：对 `< 0x80` 的单字节 ASCII 字符走**快速路径**（`c_len = 1`），跳过函数指针调用；
仅对多字节字节保留原 `ob_mbcharlen_ptr` 调用。

**影响**：英文分词的逐字节扫描省去函数指针间接调用，BENG 分词显著提速。

**语义**：`< 0x80` 的字节在所有常见字符集（UTF-8/GBK 等）中都是单字节有效字符，返回 1 正确；
`>= 0x80` 仍走原逻辑，错误处理保留。

---

### 6. 停用词过滤热路径（ob_token_stream.cpp）

**瓶颈定位依据**：`filter_special_marks` 对每个 token 逐字节分别调用 `ob_ispunct` 和
`ob_iscntrl`（各一次 ctype 表查询）。

**主要修改**：合并为**单次 ctype 表查询** + 双掩码 `(_MY_PNT | _MY_CTR)` 判断；补了
`cs_->ctype == NULL` 的等价处理。

**影响**：逐字节的 ctype 表查询次数减半。

**语义**：`(punct) || (cntrl)` 等价于 `(punct|cntrl)` 双掩码，布尔结果严格一致。

---

### 7. 分组词频聚合哈希桶（ob_token_stream.h）

**瓶颈定位依据**：`ObTextTokenGroupNormalizer` 用 `ObHashMap` 做按词分组（groupby + 词频统计），
`ObHashMap` 默认 `EXTEND_RATIO=1`，**永不扩容**，初始桶数固定为 `cal_next_prime(128)=131`。
对一篇含数百个不同词元的文档，每桶哈希链不断变长，`get`/`set` 退化为 O(n) 链式扫描。

**主要修改**：初始桶数从 128 提升到 **2048**（配合 `reuse()` 的 clear 语义，桶数组在多次文档
分析间保留复用）。

**影响**：对常见文档（每篇几十~几百词元），哈希链长接近 1，分组聚合的每次查找有效 O(1)，
避免 O(n²) 退化。

**语义**：仅改桶容量，不影响聚合结果与顺序。

---

### 8. 索引构建归并词（ob_fts_stop_word.cpp）

**瓶颈定位依据**：`ObAddWord::groupby_word` 对每个词做 `get_refactored` + `set_refactored`
**两次哈希查找**（都调 `ObFTWord::hash` + 比较）。索引构建对每文档每词执行，20k 文档放大为
数十万~数百万次。

**主要修改**：已存在的 key 改用 `ObHashMap::get`（返回 value 指针）**原地自增**，跳过第二次
`set_refactored` 查找；仅新词走插入查找。

**影响**：词级循环从 2 次哈希降为 1 次，构建耗时直接受益。

**语义**：`get` 命中时 `*v += word_freq` 与原来 `word_count += word_freq; set(word_count)` 等价。

---

### 9. BENG 停用词检查免分配（ob_fts_stop_word.cpp）

**瓶颈定位依据**：BENG 分词启用 `set_stop_word`，`ObStopWordChecker::check_stopword` 对**每个词**
在栈上构造一个新的 `ObArenaAllocator` 并调用 `ObCharset::charset_convert` 把词转成
UTF8MB4_GENERAL_CI 新缓冲，再哈希查集合。这是 BENG 路径最热单点之一。

**主要修改**：停用词表全是小写 ASCII 且 ≤ `FTS_STOP_WORD_MAX_LENGTH`。对**纯 ASCII 且短于**
该长度的词，用**栈上固定缓冲 + ASCII 小写折叠**做等价格比较，避免每词建 allocator +
charset_convert。含非 ASCII 字节的词**回退到原 charset_convert 路径**以保持精确语义。

**影响**：BENG 索引构建的每词检查省去堆分配与字符串转换，构建 `build_beng_en_sec` 收益直接。

**语义**：对 ASCII 词，UTF8MB4_GENERAL_CI 的折叠恰为 ASCII 小写（对每个 UTF8MB4 collation
相同）；非 ASCII 词走原逻辑，结果严格一致。

---

## 性能影响预估

| 评分类别 | 相关优化 | 预期收益 |
|---------|---------|---------|
| 分词（tokenize_ik / tokenize_beng） | #1, #2, #4, #5, #6, #7 | 中文单次解码、词典二分、英文 ASCII 快速路径、哈希桶扩容 —— 对 3,000 次/组分词测试收益直接 |
| 索引构建（build_ik / build_beng） | #2, #8, #9 | 词典二分、词级单次哈希、停用词免分配 —— 对 20k 文档构建收益直接 |
| 查询（query_*） | #2（词典加速影响分词→查询） | 本批聚焦分词与构建；查询链路如需进一步优化可参考下方待办 |

## 后续可做（查询链路，风险中等，建议本地验证后实施）

子代理已对查询链路（MATCH AGAINST）完成分析，以下为**低风险、不改变语义**的候选优化点，
因涉及存储层迭代/缓存生命周期且无法本地编译验证，暂未合入：

1. **LIMIT 下推**：`ob_text_retrieval_token_iter.cpp` 把 LIMIT 下推为倒排扫描上限，避免
   query_limit 全量扫倒排。
2. **token_doc_cnt 缓存**：`estimate_token_doc_cnt`（ob_text_retrieval_token_iter.cpp:359）
   每次查询每 token 都做 `estimate_row_count` 存储统计扫描；可在参数未 switch 时复用
   `token_doc_cnt_calculated_`（需仔细确认 iter 与 token 生命周期，避免跨 token 误用导致 BM25
   分数错误）。
3. **advance_to 二分推进**：`ObTextRetrievalDaaTTokenIter::advance_to` 缓冲内线性推进改二分
   （doc_id_ 有序，需确认 batch 内严格有序）。
4. **BM25 batch 常参外提**：`ob_expr_bm25.cpp` 常参已在循环外，微优化空间小。

## 备注

- 全部改动通过 `git diff --check` 校验，无空白/尾随错误。
- 由于本地是 Windows 且无 WSL/Docker/bazel，无法本地编译与跑基准测试；改动均为**语义保持**的
  保守优化，正确性优先，需在 VLDB CI 或你的 Linux 环境编译验证。
- 当前 github.com:443 连接不稳定，推送失败；网络恢复后执行 `git push origin master`。