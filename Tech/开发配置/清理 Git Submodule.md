---
tags:
  - Linux
  - 运维
---

> 使用 DeepSeek-V3 生成

# 彻底清除 Git Submodule 的步骤

要彻底清除一个 Git submodule，需要执行以下步骤：

## 1. 删除 submodule 相关配置和文件

```bash
# 1. 删除 submodule 目录
git submodule deinit -f path/to/submodule
rm -rf .git/modules/path/to/submodule
git rm -f path/to/submodule
```

## 2. 清理 .gitmodules 文件

```bash
# 编辑 .gitmodules 文件，删除对应的 submodule 配置
vim .gitmodules
# 或者使用 sed 命令自动删除
sed -i '/\[submodule "path\/to\/submodule"\]/,+3d' .gitmodules
```

## 3. 提交变更

```bash
git add .gitmodules
git commit -m "Remove submodule path/to/submodule"
```

## 4. 额外清理（可选）

```bash
# 如果 submodule 目录仍然存在（未被 git rm 删除）
rm -rf path/to/submodule

# 更新 git 索引
git config -f .git/config --remove-section submodule.path/to/submodule 2>/dev/null
```

## 注意事项

1. 如果 submodule 有未提交的修改，`deinit` 命令会失败，需要先提交或放弃这些修改
2. 在 Windows 上，可能需要使用不同的路径分隔符（反斜杠 \）
3. 如果遇到权限问题，可能需要使用 `sudo`（不推荐，最好修复权限问题）

完成以上步骤后，submodule 将被完全从你的仓库中移除，包括其历史记录和配置。