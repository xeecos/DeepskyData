# DeepskyData 数据结构

## 概览

本仓库托管 2799 个深空天体的预生成缩略图，供客户端在线拉取并本地缓存。
图像在构建期由 `scripts/gen-dso-repo.py` 生成，取图与后处理规则复用
`gen-dso-thumbs.py`（与 App 内置兜底包同源，保证构图一致）。

- 分辨率：**400 × 400 px**，JPEG quality 70（单档，与内置包一致）
- 视场：`clamp(视径 × 1.9, 8′, 240′)`，天体约占画面一半
- 图像源：
  - **PS1 彩色**：星系/星团等（PanSTARRS1 gri 波段合成，红=i 绿=r 蓝=g）
  - **DSS2 红片**：发射星云/SNR/暗星云、dec≤−29.5°、视径>25′
- 署名：DSS2 © STScI/NASA · PanSTARRS1 © STScI

## 文件布局

```
DeepskyData/
├── catalog.json        # 全量星表（2799 条）
├── manifest.json       # 图片清单（含 sha256/bytes/rev）
└── images/
    ├── 0/              # 16 桶：sha256(id) 首字符 0-f
    ├── 1/
    └── ... f/
        └── M51.jpg     # 文件名 = sanitize_id(id) + ".jpg"
```

`sanitize_id` 规则：非 `[A-Za-z0-9]` 字符替换为 `_`
（`Sh2-170` → `Sh2_170`、`PK118+02.1` → `PK118_02_1`）。

## catalog.json

```json
{
  "schema": 1,
  "generated_at": "2026-11-18T03:19:00Z",
  "count": 2799,
  "objects": [
    {
      "id": "M51",
      "name": "涡状星系",
      "ra": 202.4696,
      "dec": 47.1953,
      "size_arcmin": 11.2,
      "class": "Galaxy",
      "source": "ps1",
      "fov_arcmin": 21.28
    }
  ]
}
```

`id` 使用客户端星表的稳定编号（M31 / NGC7000 / Sh2-170），
不是 README 里的 M001/C001 展示编号。

## manifest.json（增量核心）

```json
{
  "schema": 1,
  "version": "2026-11-18.0319",
  "generated_at": "2026-11-18T03:19:00Z",
  "resolution": 400,
  "format": "jpeg",
  "quality": 70,
  "base_url": "https://cdn.jsdelivr.net/gh/xeecos/DeepskyData@main",
  "fallback_url": "https://raw.githubusercontent.com/xeecos/DeepskyData/main",
  "count": 2799,
  "files": [
    {
      "id": "M51",
      "path": "images/a/M51.jpg",
      "bytes": 31284,
      "sha256": "9f2a...",
      "rev": 1
    }
  ]
}
```

### 增量更新机制

- **生成器增量**：`--incremental` 只处理高分缓存里缺失的天体；字节未变时
  `sha256` 不变、`rev` 不增。
- **客户端增量**：本地缓存记录每张图的 sha256（`.sha` 侧车）；拉取 manifest
  后只下载「本地没有」或「sha256 变了」的图。manifest 本身用 ETag
  `If-None-Match` 条件请求，未变更返回 304。
- **校验**：客户端下载后校验 JPEG 魔术字节 + sha256，不符则换源重试。

## 下载端点

| 用途 | URL |
|---|---|
| 清单（国内主） | `https://cdn.jsdelivr.net/gh/xeecos/DeepskyData@main/manifest.json` |
| 清单（兜底） | `https://raw.githubusercontent.com/xeecos/DeepskyData/main/manifest.json` |
| 图片 | `{base_url}/{path}` |

> 注意：本仓库**不使用 Git LFS**。实测 jsDelivr / raw 对 LFS 文件只返回
> 指针文本而非图片，国内会全部失效。所有图片以普通 Git 文件存储。

## 重新生成

见主仓库 `docs/dso-repo-plan.md` 与 `scripts/gen-dso-repo.py`：

```bash
python scripts/gen-dso-repo.py --all --jobs 8       # 全量（缺失的联网取）
python scripts/gen-dso-repo.py --incremental        # 增量
python scripts/gen-dso-repo.py --repack             # 只从缓存重压
python scripts/gen-dso-repo.py --verify             # 全量 sha256 校验
```
