# rdkafka-sys (sufar fork)

> **为什么这个仓库存在 / Why this repo exists**
>
> 本仓库 = **crates.io rdkafka-sys 4.10.0** + **vendored librdkafka 2.15.1** + **WITH_SNAPPY cmake patch**。
> 它是 [kafka-manager](https://github.com/sufar/kafka-manager) 的 `[patch.crates-io]` 依赖。
>
> This is a fork of crates.io `rdkafka-sys` 4.10.0 with the vendored librdkafka
> C library bumped to 2.15.1 and a one-line build.rs patch that explicitly
> enables `WITH_SNAPPY` for cmake builds. It exists to be consumed as a
> `[patch.crates-io]` git dependency by kafka-manager.

---

## 与上游的三点差异

### 1. vendored librdkafka：2.12.1 → 2.15.1

源码树 `librdkafka/` 替换为官方 [v2.15.1](https://github.com/confluentinc/librdkafka/releases/tag/v2.15.1)
release tarball（SHA256 `23c8575c7d1ced07246cb9cf200c11325b72201fd4134a02414ca869fbdd8ed3`，已校验），无源码级修改。

关键修复（与本项目直接相关）：

- **2.15.0**：CMake 构建下 `rd_atomic32/64_set` 使用非原子回退，导致 `ALL_BROKERS_DOWN` 事件不触发（本项目正是 cmake 构建）；KIP-848 心跳错误处理。
- **2.14.2**：timer 数据竞争、consumer close 时 `LeaveGroup` 段错误、`ListConsumerGroups` 重复。
- **2.14.2 / 2.15.1**：捆绑 OpenSSL / libcurl / zstd / zlib / cJSON 大量 CVE 修复。

`src/bindings.rs` 维持 4.10.0 预生成版本：librdkafka ABI 后向兼容，2.15.x 新 API（share consumer 等）不暴露，本项目也用不到。

### 2. WITH_SNAPPY cmake patch（上游未修，必须保留）

`build.rs` 的 cmake 构建路径显式 `config.define("WITH_SNAPPY", "1")`。

原因：Windows 上 librdkafka 的 cmake 构建默认 `WITHOUT_WIN32_CONFIG=ON`（跳过 `win32_config.h`），
此时 `WITH_SNAPPY` 编译宏来自同名 cmake 变量，而**没有任何地方设置它** → 默认编译为
`WITH_SNAPPY=0` → snappy 支持被裁掉 → 消费 snappy 压缩消息时报运行时错误 `"Not implemented"`。
（Linux/macOS 走 `packaging/cmake/config.h.in`，其中硬编码 `#define WITH_SNAPPY 1`，不受影响。）

上游状态（截至 2026-10 核实）：librdkafka 2.15.1 与 rdkafka-sys master 均未修。
完整 diff 见仓库根目录 [`WITH_SNAPPY-cmake-build.patch`](./WITH_SNAPPY-cmake-build.patch)。

### 3. 预编译二进制支持（本地新增，跳过 C 库编译）

`build.rs` 新增：设置环境变量 **`LIBRDKAFKA_PREBUILT_DIR`** 指向包含预编译静态库的目录时，
完全跳过 librdkafka 的 cmake 编译（约 2 分钟 → 秒级）：

```bash
# 目录内容：Unix 为 librdkafka.a，MSVC 为 rdkafka.lib
export LIBRDKAFKA_PREBUILT_DIR=/path/to/prebuilt
cargo build
```

预编译产物在本仓库 [Releases](../../releases) 中（`librdkafka-<target>.tar.gz` + `.sha256`），
由 [.github/workflows/prebuilt.yml](./.github/workflows/prebuilt.yml) 构建。

**约束（违反会在链接/运行期报错）**：

- 预编译库的 feature 集固定为：**zlib + zstd + lz4-ext + snappy，无 SSL/SASL/CURL**
  （与 kafka-manager 的 rdkafka feature 集一致）。压缩库本身（libz-sys/zstd-sys/lz4-sys）
  仍由 cargo 正常编译链接，预制的只有 librdkafka。
- Linux 产物与构建机 glibc 绑定：CI 产物基于 ubuntu-22.04（glibc 2.35，x86_64 与 aarch64）；
  早期手动上传的本地产物（aarch64）要求 glibc ≥ 2.43，文件名带 `-glibc2.43` 标记，二者都在 Release 中，按机器选择。

## 开源协议 / License

本仓库各部分沿用上游原有协议（详见 [NOTICE.md](./NOTICE.md)）：

| 组成部分 | 来源 | 协议 |
|---|---|---|
| rdkafka-sys（绑定层，含本仓库修改） | [rust-rdkafka](https://github.com/fede1024/rust-rdkafka) | **MIT**（[LICENSE](./LICENSE)，上游版权归 Federico Giraud，修改部分归 sufar） |
| librdkafka（vendored，未修改） | [librdkafka v2.15.1](https://github.com/confluentinc/librdkafka) | **BSD 2-Clause**（[librdkafka/LICENSE](./librdkafka/LICENSE)） |
| librdkafka 捆绑第三方组件 | 见 [librdkafka/LICENSES.txt](./librdkafka/LICENSES.txt) | 各自原有协议（原样保留） |

预编译产物压缩包内附带全部协议文本（`licenses/` 目录），满足 BSD 2-Clause 对二进制再分发的保留要求。

## 引用方式（kafka-manager 的 Cargo.toml）

```toml
[patch.crates-io]
rdkafka-sys = { git = "https://github.com/sufar/rdkafka-sys", tag = "v4.10.0+2.15.1" }
```

`+` 后缀版本号规则：`4.10.0+x.y.z` 中的 `x.y.z` 必须等于 vendored librdkafka 的实际版本
（`tests/version_check.rs` 会断言二者一致）。

## 将来如何升级 librdkafka

1. 下载新 release tarball，校验 SHA256，整树替换 `librdkafka/`。
2. `Cargo.toml` + `Cargo.toml.orig` 的版本号改为 `4.10.0+<新版本>`；`changelog.md` 加 local 条目。
3. **不得丢失** build.rs 的 WITH_SNAPPY patch 和 `LIBRDKAFKA_PREBUILT_DIR` 支持。
4. `cargo test`（version_check）+ 下游 kafka-manager 的 `rdkafka_version` 测试验证。
5. 打新 tag（`v4.10.0+<新版本>`），kafka-manager 更新 patch 指向；如 feature 集不变，
   重跑 prebuilt workflow 出新二进制。

上游 rdkafka-sys 出新版本（如 4.11.x）时同理：rebase 本仓库两个本地改动（librdkafka 版本、build.rs 两个 patch）。

---

## English summary

- Fork of crates.io **rdkafka-sys 4.10.0**; vendored **librdkafka 2.15.1** (was 2.12.1).
- `build.rs` defines `WITH_SNAPPY=1` for cmake builds — fixes snappy decompression
  (`Not implemented` runtime error) on Windows, where librdkafka's cmake build
  skips `win32_config.h` (`WITHOUT_WIN32_CONFIG=ON`) and nothing else defines it.
  **Not fixed upstream** as of librdkafka 2.15.1 / rdkafka-sys master.
  See [`WITH_SNAPPY-cmake-build.patch`](./WITH_SNAPPY-cmake-build.patch).
- `LIBRDKAFKA_PREBUILT_DIR=/path` skips compiling librdkafka by linking a prebuilt
  static archive from GitHub Releases (feature set: zlib+zstd+lz4-ext+snappy, no SSL).
- Consumed via `[patch.crates-io]` git dependency pinned to a tag.

The remainder of this README is the original upstream one; see
[changelog.md](./changelog.md) for the local change entries.

---

# rdkafka-sys (upstream README)

Low level bindings to [librdkafka](https://github.com/confluentinc/librdkafka),
the Apache Kafka C/C++ client library. For the safe Rust wrapper see
[rdkafka](https://crates.io/crates/rdkafka).
