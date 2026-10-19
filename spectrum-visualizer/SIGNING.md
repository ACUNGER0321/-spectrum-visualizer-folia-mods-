# 签名入库流程

> 本目录**不**包含 `folium.sig.json`。一旦把模组放到
> [folium-compound](https://github.com/chthollyphile/folium-compound) 让维护者用官方密钥签一次，
> 该 JSON 会由工具自动生成并落到原位。
>
> 你要做的是先把目录完整推到 folium-compound 的 `mods/community/spectrum-visualizer/`，
> 再走"作者提交 → 维护者签名"流程。

---

## 三步走

### 1. 准备 `preview.jpg`

| 项 | 要求 |
|---|---|
| 格式 | PNG / JPG / WebP |
| 尺寸 | 推荐 1280 × 720，至少 640 × 360 |
| 大小 | ≤ 1 MB |
| 命名 | `preview.jpg`（清单 `preview` 字段指向它） |

### 2. 推到 `folium-compound` 的 `mods/community/`

```bash
mkdir -p mods/community/spectrum-visualizer
cp mods/spectrum-visualizer/{mod.json,client.mjs,preview.jpg} \
   mods/community/spectrum-visualizer/
```

并给 `community.json` 加一条（按其它 community 模组格式）：

```json
"spectrum-visualizer": {
  "owner": "<your-github-handle>",
  "source": "https://github.com/<you>/spectrum-viewer",
  "subdir": "mods/spectrum-visualizer",
  "ref": "<commit-sha>",
  "version": "0.1.0"
}
```

### 3. 提 issue 让维护者签名

到 [folium-compound/issues/new](https://github.com/chthollyphile/folium-compound/issues/new) 开
issue（标题按模板，如 `Mod submission: spectrum-visualizer v0.1.0`），body 填 fork 子目录、
commit SHA、版本号，打 `mod-submission` 标签。

`submission-check.yml` 自动校验（mod.json 字段、无符号链接、无占位签名、体积），然后维护者
留 `/sign <key>`，`sign.yml` 用 CI 密钥对模组目录做 SHA-256 + Ed25519 签名，回写
`folium.sig.json` 并更新 `index.json`。

---

## 本地开发者签名（可选，仅调试用）

> ⚠️ 不要把本地私钥签出的 `folium.sig.json` 推到共享仓库；CI 发现不合规签名会直接下架。

```bash
cd <folium-compound 根目录>
npm run verify -- --strict --list mods/community/spectrum-visualizer   # dry-run
npm run sign   -- --key ~/.config/folium-signing/<key>.key.json \
                  mods/community/spectrum-visualizer                   # 本地签名
```

---

## 不签名也能本地加载

可以。Folia 主播放器本地 / 开发者模式不强制校验签名，但发布到 folium-compound 交付真实用户
时必须签名，否则 Folia 会标"未验证"并要求额外确认。

---

## 小提醒

- 改动 `mod.json` / `client.mjs` / 预览图都会让签名 digest 失效 → 升级版本号重走流程。
- `community.json` 的 `ref` 锁 fork 仓库的具体 commit，更新模组时指向新 commit 再签一次。
