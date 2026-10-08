# Agent Note: 日文补上常用罗马字拼法，完整读音的平假名留在第一页

Status: implemented

## Problem

用户拿 Rime（雾凇加カギロイ）对比日文输入时报了两个问题：

- `xtu` 打不出促音っ。罗马字表照搬 `romaji_converter.cpp`，只有 `xtsu`/`ltsu`，`xtu` 整段停在待定字母里，候选为空。同一张表还缺微软和 Google 日文输入法都认的 `zya`/`zyu`/`zyo`、`dya`/`dyu`/`dyo`。
- `tyou` 打不出ちょう。转换本身没错（`tyou`、`chou` 都转成ちょう），但词库里没有ちょう这个假名词条，平假名被追加在全部汉字和联想之后，排在第 23 个；有假名词条的きょう、にほん本来就在第二位。

## Decision

- `ROMAJI_TABLE` 补上 `xtu`、`ltu`、`zya`、`zyu`、`zyo`、`dya`、`dyu`、`dyo`。反查 `hiragana_to_romaji` 仍按「最长拼法、再字母序」取一个：っ 还是 `xtsu`，じゃ 还是 `jya`，金样里记录的拼法不变。
- `JapaneseProvider` 在读音完整（没有待定字母）时，把平假名挪到不晚于第二位（`KANA_SLOT = 1`），其余行保持相对顺序；首位仍是最可能的转换。还挂着半截字母时不挪，那时该先给能拼成的词。片假名位置不变。

## Alternatives considered

- **平假名放第一位**：Google 日文输入法在变换前就是平假名，第一位最接近它。不用是因为水杉的候选条第一位就是空格上屏的那个，放平假名会让 `nihon` 空格上屏にほん而不是日本，改变了所有人的日常输入。
- **按权重重排整张列表**：平假名的 `KANA_WEIGHT` 本来最高，按权重排它会到第一位，问题同上；而且这个列表的顺序是有意按来源拼起来的（见 `query_into` 的注释），整体重排会打乱联想与转换的先后。
- **只补 `xtu`，其余拼法不动**：改动最小。但 `zya`、`dya` 和 `xtu` 是同一类缺口（主流输入法都认、C++ 表缺），用户换个词就会再撞上。没有补 `xka`/`xke`（小写ヵヶ）和 `lwa`：前者在平假名里没有通用写法，后者会把ゎ的反查从 `xwa` 变成 `lwa`。

## 未决：`nn` 的规则

用户还报了「`nn` 打出んん」。引擎里 `n`、`nn`、`honn`、`sinnbunn`，选候选、回车上屏读音、上屏原文都得到ん，没有复现。现行规则和微软、Google 输入法有一处不同：`nn` 后面跟元音或 `y` 时只把第一个 n 当ん（`konnichiha` 得こんにちは，`sinnyou` 得しんにょう），主流输入法则一律把 `nn` 当ん（`sinnyou` 得しんよう，`konnichiha` 得こんいちは）。改不改要产品决定，这里不动。

## Consequences

- **收益**：习惯微软、Google 拼法的用户能打出っ和じゃ行、ぢゃ行；没有假名词条的读音，平假名也在第一页。
- **代价**：完整读音时第二位固定给平假名，原来排第二的转换往后挪一位。
- **没有覆盖**：`nn` 的规则差异；桌面宿主上的实际效果只靠引擎测试推断。

## Verification

- `cargo test -p msime-engine`：1301 条单测和 31 条金样全部通过。新测试 `microsoft_and_google_spellings`、`a_complete_reading_keeps_its_hiragana_on_the_first_page`；后者在去掉挪位时失败。
- 用 `MSIME_EVAL_RESOURCES` 指向的真实日文词库：`tyou`、`chou` 的ちょう在第二位；`real_model_answers_common_readings` 通过。`real_model_loads` 失败是因为它断言 dict-v2.0.1 的词条数（1,284,987），本地词库是更新的版本（1,285,198），与这次改动无关。
