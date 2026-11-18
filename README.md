# DeepskyData · 深空天体图像库

> 面向「口袋天文台」App 的云端图像数据仓库。
> 数据集版本：`2026-11-18.0319` · 更新于 2026-11-18。

本仓库托管 **2799 个深空天体的预生成缩略图**，供客户端在线拉取并本地缓存。
图像在构建期由 `scripts/gen-dso-repo.py` 生成，取图与后处理规则与 App 内置
兜底包完全同源，保证构图一致。

## 数据集一览

| 项 | 值 |
|---|---|
| 天体数 | **2799** |
| 分辨率 | **400 × 400 px**（单档，与内置包一致） |
| 格式 | JPEG quality 70 |
| 总体积 | 82.3 MB（图片）+ 0.4 MB（manifest）+ 0.5 MB（catalog） |
| 图像源 | PanSTARRS1 (STScI) · DSS2 (STScI/NASA) |
| 视场 | `clamp(视径 × 1.9, 8′, 240′)`，天体约占画面一半 |

## 快速使用

### 国内推荐（jsDelivr CDN）

```text
清单：https://cdn.jsdelivr.net/gh/xeecos/DeepskyData@main/manifest.json
图片：{manifest.base_url}/{manifest.files[i].path}
```

`manifest.json` 里已经写好了 `base_url`（jsDelivr）和 `fallback_url`
（raw.githubusercontent）。国内实测：清单 4.9s、单图 1.3s。

### 兜底通道

```text
https://raw.githubusercontent.com/xeecos/DeepskyData/main/manifest.json
```

> 国内直连 raw 基本不可用（20s 超时），仅作 jsDelivr 故障时的兜底。
>
> **本仓库不使用 Git LFS**：实测 jsDelivr / raw 对 LFS 文件只返回指针文本
> 而非图片，所有图片均以普通 Git 文件存储。

## 数据结构

```text
DeepskyData/
├── README.md          本文件
├── schema.md          数据结构详细说明
├── catalog.json       全量星表（2799 条：id / 坐标 / 分类 / 视场）
├── manifest.json      图片清单（sha256 / bytes / rev，增量核心）
└── images/
    ├── 0/             16 桶分片：sha256(id) 首字母 0-f
    ├── 1/
    └── ... f/
        └── M51.jpg    文件名 = sanitize_id(id) + ".jpg"
```

`id` 使用客户端星表的稳定编号（`M31` / `NGC7000` / `Sh2-170`），
`sanitize_id` 把非字母数字字符替换为 `_`（`Sh2-170` → `Sh2_170`）。

### manifest.json 示例

```json
{
  "schema": 1,
  "version": "2026-11-18.0319",
  "resolution": 400,
  "format": "jpeg",
  "base_url": "https://cdn.jsdelivr.net/gh/xeecos/DeepskyData@main",
  "count": 2799,
  "files": [
    {
      "id": "M51",
      "path": "images/4/M51.jpg",
      "bytes": 21900,
      "sha256": "f3b89545...",
      "rev": 1
    }
  ]
}
```

完整字段说明见 [`schema.md`](schema.md)。

## 增量更新

- **生成端**：`--incremental` 只处理高分缓存里缺失的天体；字节未变时 sha256
  不变、rev 不增。
- **客户端**：本地缓存记录每张图的 sha256，拉取 manifest 后只下载「本地没有」
  或「sha256 变了」的图；manifest 本身用 ETag `If-None-Match` 条件请求，
  未变更返回 304。
- **校验**：下载后校验 JPEG 魔术字节 + sha256，不符则换源重试。

重新生成（见主仓库 `docs/dso-repo-plan.md`）：

```bash
python scripts/gen-dso-repo.py --all --jobs 8       # 全量（缺失的联网取）
python scripts/gen-dso-repo.py --incremental        # 增量
python scripts/gen-dso-repo.py --repack             # 只从缓存重压
python scripts/gen-dso-repo.py --verify             # 全量 sha256 校验
```

## 数据来源与署名

- **PanSTARRS1 (PS1)**：星系、星团等；红=i / 绿=r / 蓝=g 彩色合成
- **DSS2 红片**：发射星云、超新星遗迹、暗星云、深南天（dec ≤ −29.5°）、
  视径 > 25′ 的大天体

```text
DSS2 © STScI/NASA · PanSTARRS1 © STScI
```

数据仅作个人观星参考，不分发、不商用。

## 校验状态

- 2799 / 2799 张图通过 `--verify` 全量 sha256 校验
- jsDelivr 端到端实测：清单 + 图片下载，sha256 与 manifest 一致
- 仓库总体积 83MB，低于 jsDelivr 150MB 默认上限
