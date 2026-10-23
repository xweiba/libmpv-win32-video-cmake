# PaoPao 补丁清单（Windows / llvm-mingw clang）

本仓库 = Predidit/libmpv-win32-video-cmake `master` + 下列 PaoPao 统一内核补丁。
补丁由 `packages/ffmpeg.cmake`、`packages/mpv.cmake` 的
`git apply ${CMAKE_CURRENT_SOURCE_DIR}/<pkg>-*.patch` 按文件名顺序整体应用。

基线（跟随上游）：

| 组件 | 版本 |
| --- | --- |
| FFmpeg | `5b9a3ad71a4b54979e7afc3a1a4269763c4f41fc`（主线） |
| mpv | `0b7ed670f7c353dd3dd4f8ae0fc788a181a15aa6`（主线） |

## FFmpeg（`packages/ffmpeg-*.patch`，应用顺序即文件名顺序）

| 文件 | 来源 | 内容 |
| --- | --- | --- |
| `ffmpeg-hls-kazumi-combined.patch` | 上游 Predidit（已是 PTS 偏移版） | `hls_ad_filter` PTS 偏移去广告（`pts_offset`、`ad_start_*/ad_resume_*_pts`、`video/audio_resumed`、`last_main_audio_pts`、-2s/+10s 恢复窗口、seek 后 `after_seek` 复位）；`seg_allow_img` PNG/GIF 伪装分片探测；`hls_probe` 非标扩展名返回 `AVPROBE_SCORE_MAX/2`。 |
| `ffmpeg-segmented-custom-io.patch` | PaoPao（移植自 libmpv-android-video-build `ffmpeg-segmented-custom-io.patch`） | HLS/DASH 新增 `allow_custom_io`（bool，默认 0）。未知协议仅在开启且 URL 带非空 `scheme:` 时经 `s->io_open` 打开，否则 `AVERROR_INVALIDDATA`；`proto_name` 为空时 `proto_name_len`/`is_http` 空指针修复。FFmpeg 主线 DASH 的 `open_url`/`parse_manifest` 已统一走 `s->io_open` 并以 `ff_format_io_close` 关闭 manifest，因此只需补协议放行逻辑。 |
| `ffmpeg-segmented-io-cancel.patch` | PaoPao（libmpv-darwin-build） | HLS/DASH `open_url` 开头 `ff_check_interrupt()` 命中即返回 `AVERROR_EXIT`。 |

## mpv（`packages/mpv-*.patch`）

| 文件 | 来源 | 内容 |
| --- | --- | --- |
| `mpv-0001-dxd11-render-api.patch` | 上游 Predidit | `MPV_RENDER_API_TYPE_DXGI` 渲染 API（`render_dxgi.h`）。 |
| `mpv-0002-restore-tls-verify-default.patch` | 上游 Predidit (#7) | mpv 主线把 `tls-verify` 默认改为 yes，此补丁恢复旧默认，保持行为一致。 |
| `mpv-nested-stream-callback.patch` | PaoPao（Android `nested-stream-callback.patch` 与 Darwin `mpv-nested-stream-callback.patch` 的并集） | 见下。 |

`mpv-nested-stream-callback.patch`（`demux/demux_lavf.c`、`stream/stream.h`、`stream/stream_cb.c`）：

- `struct nested_stream` 增加 `stream`、`stream_cb`；`nested_stream_read`（0 字节 → `AVERROR_EOF`）、`nested_stream_seek`（`AVSEEK_SIZE`/`SEEK_END`/`SEEK_CUR`）。
- `nested_stream_cb_is_allowed`：协议须经 `mp_streamcb_lookup` 注册且在 `protocol_whitelist` 内（`av_match_name`）。
- `nested_stream_cb_open`：打开前后 `mp_cancel_test`（命中 `AVERROR_EXIT`），`stream_create(url, STREAM_READ | stream_origin, demuxer->cancel, global)`，`avio_alloc_context` + seekable，登记到 `priv->nested`。
- `nested_io_open`：非写打开且允许时走回调（先于 mpv 主线新增的 libcurl 后端）；顶层（`!s->pb`）成功时设 `AVFMT_FLAG_CUSTOM_IO`。
- `nested_io_close2`：`pb == priv->pb` 直接返回；回调条目关闭后保留墓碑（释放 stream 与 buffer），重复 close 不会落到 `avio_close`。
- `demux_close_lavf`：`avformat_close_input` 前若 `avfc->pb` 是回调条目则设 `AVFMT_FLAG_CUSTOM_IO`；之后用 `nested_stream_cb_free` 释放全部回调条目（先 `stream_cb_detach_cancel`，再 free stream、buffer、`avio_context_free`）。
- `stream_cb_detach_cancel()`（stream_cb.c 新增）：`mp_cancel_set_parent(p->cancel, NULL)`，避免 `cancel_destroy` 的 `!c->slaves.head` 断言。

未加补丁：

- `mpv-filter-metadata-key-type.patch`：mpv `0b7ed670` 的 `player/command.c` `mp_property_filter_metadata` 已包含等价修复（`remaining || ka->action != M_PROPERTY_GET_TYPE`），无需重复。

## FFmpeg configure 追加能力（`packages/ffmpeg.cmake`）

- 滤镜：`overlay equalizer astats aresample aformat anull asplit pan bandpass amerge`
- 原声缓存导出：`--enable-muxer=mp4 --enable-muxer=mov --enable-bsf=aac_adtstoasc`

## 打包（`packages/mpv-packaging.cmake`）

- `mpv-dev-*` 开发包在所有架构上都打成 zip（上游 x86_64 为 7z），供 media-kit 以 `cmake -E tar` 解压。

## 验证

- 本地：所有补丁对上述 FFmpeg/mpv 提交 `git apply --check` 干净应用；`hls.c`/`dashdec.c` 与 mpv `demux_lavf.c`/`stream_cb.c` 本机编译无告警，libmpv 链接成功。
- CI：`.github/workflows/llvm_clang.yml`（工具链）→ `mpv_clang.yml`（x86_64 / x86_64_v3 / i686 / aarch64，`github_release=true` 发布 `YYYYMMDDhhmm` tag）。
- 产物：`strings libmpv-2.dll` 应包含 `allow_custom_io`、`seg_allow_img`、`hls_ad_filter`、`astats`、`aac_adtstoasc`；导出表含 `mpv_stream_cb_add_ro`、`mpv_render_context_create`。
