# NOTICE / 协议声明

本仓库（sufar/rdkafka-sys）是以下开源软件的组合与修改，各组成部分沿用其原有协议：

## 1. rdkafka-sys — MIT License

- 来源 / Source: https://github.com/fede1024/rust-rdkafka （crates.io `rdkafka-sys` 4.10.0）
- 协议 / License: **MIT**，Copyright (c) 2016 Federico Giraud（见仓库根目录 [LICENSE](./LICENSE)）
- 本仓库的修改（librdkafka 版本升级、build.rs 的 WITH_SNAPPY 与 LIBRDKAFKA_PREBUILT_DIR
  patch、CI 配置等）：Copyright (c) 2026 sufar，**同样以 MIT 协议发布**。
- 注：上游 crates.io 打包未附带 LICENSE 文件，本仓库按 MIT 条款要求补齐。

## 2. librdkafka — BSD 2-Clause License

- 来源 / Source: https://github.com/confluentinc/librdkafka v2.15.1
- 协议 / License: **BSD 2-Clause**，Copyright (c) 2012-2022 Magnus Edenhill, 2023 Confluent Inc.
  （见 [librdkafka/LICENSE](./librdkafka/LICENSE)）
- 本仓库对 librdkafka 源码树**未做任何修改**（原样 vendor 官方 release tarball）。

## 3. librdkafka 捆绑的第三方组件

`librdkafka/` 内含的第三方组件协议文本全部原样保留在
[librdkafka/LICENSES.txt](./librdkafka/LICENSES.txt) 及 `librdkafka/LICENSE.*` 系列文件中
（snappy、lz4、cJSON、crc32c、fnv1a、hdrhistogram、murmur2、nanopb、opentelemetry、
pycrc、queue、regexp、tinycthread、wingetopt 等），其各自协议不变。

## 4. 预编译二进制（GitHub Releases 产物）

Releases 中的 `librdkafka-<target>.tar.gz` 为上述 librdkafka 的预编译静态库，
每个压缩包内均附带上述协议文本（`licenses/` 目录），以满足 BSD 2-Clause
第 2 条"以二进制形式再分发时须在分发材料中保留版权声明、条件列表与免责声明"的要求。

**下游使用者注意**：若你的应用链接并分发上述静态库（无论自编译还是预编译），
你需要在自己的分发物中同样保留 librdkafka 及其第三方组件的版权声明
（BSD 2-Clause 要求）；rdkafka-sys 绑定层部分按 MIT 保留根目录 LICENSE 即可。
