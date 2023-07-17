---
title: Git Hook 检查
tags: 
  - Git
date: 2022-08-24
categories:
  - 后端
---

### 设置全局的git hook
```shell
mkdir ~/.githooks
git config --global core.hooksPath ～/.githooks
cd ~/.githooks
touch pre-commit
```

### pre-commit 文件内容
```shell
#!/bin/zsh
files=$( git diff --cached --name-only --diff-filter=ACMR |xargs -0 perl -CDS -lne 'print "$ARGV: $., $_" if /\p{Han}/;')
if [[ -n $files ]]; then
    echo "The following files contain Chinese characters:"
    echo "$files"
    /bin/echo 'please use [\u4e00-\u9fa5] to check Chinese characters'
    exit 1
fi

exit 0
```

### 忽略 pre-commit
`git commit -m "xxx" --no-verify`