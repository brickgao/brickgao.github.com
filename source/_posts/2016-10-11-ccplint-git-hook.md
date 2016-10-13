title: 利用 git hook 在 git commit 之前用 cpplint 自动检查代码
date: 2016-10-11 23:50:00
tags:
- C++
categories:
- Develop
---

最近主要在写 C++，为了防止自己手滑，需要一个自动化检查代码的方法。而由于没有 CI 可以用，姑且采用了利用 Git Hook 在 `git commit` 之前检查一遍代码的方案。

<!-- more -->

### 获取 cpplint

通过 git 来 clone 下 google 的 `styleguide`：

```bash
$ git clone https://github.com/google/styleguide.git styleguide
```

配置 `cpplint` 到系统内：

```bash
$ sudo mkdir /opt/cpplint
$ sudo cp styleguide/cpplint/cpplint.py /opt/cpplint/
$ sudo chmod +x /opt/cpplint/cpplint.py
$ sudo ln -s /opt/cpplint/cpplint.py /usr/bin/cpplint
```

这时候在终端里输入 `cpplint` 应该可以获取到 `cpplint` 的提示了，到此步 `cpplint` 已经配置完毕。

### 对某个 git 工作目录增加 git hook 来检查代码风格

在 git 工作目录的顶层，即是有 `.git` 文件夹的目录来配置 git hook，请务必确保之前没有 pre-commit 的 hook：

```bash
$ curl -fsSL https://git.io/vPBpe > ./.git/hooks/pre-commit
$ chmod +x ./.git/hooks/pre-commit
```

这样在每次 `git commit` 之前，`cpplint` 都会自动检查代码风格，如果检查不通过，就不会进行提交。

对于具体的 pre-commit 的 hook 脚本，简单来说就是找到这次修改过的文件，对每个修改过的文件进行 `cpplint` 的检查，如果检查到有错误就返回一个错误代码，从而终止 `git commit` 操作。

```sh
#!/bin/sh
#
# Modified from http://qiita.com/janus_wel/items/cfc6914d6b7b8bf185b6
#
# An example hook script to verify what is about to be committed.
# Called by "git commit" with no arguments.  The hook should
# exit with non-zero status after issuing an appropriate message if
# it wants to stop the commit.
#
# To enable this hook, rename this file to "pre-commit".

if git rev-parse --verify HEAD >/dev/null 2>&1
then
    against=HEAD
else
    # Initial commit: diff against an empty tree object
    against=4b825dc642cb6eb9a060e54bf8d69288fbee4904
fi

# Redirect output to stderr.
exec 1>&2

cpplint=cpplint
sum=0
filters='-build/include_order,-build/namespaces,-legal/copyright,-runtime/references'
        
# for cpp
for file in $(git diff-index --name-status $against -- | grep -E '\.[ch](pp)?$' | awk '{print $2}'); do
    $cpplint --filter=$filters $file
    sum=$(expr ${sum} + $?)
done
    
if [ ${sum} -eq 0 ]; then
    exit 0
else
    exit 1
fi
```

### 常见问题

#### 虽然 cpplint 检查不过，但是我想提交怎么办？

增加 `-n` 参数，但不推荐提交检查不通过的代码。

```bash
$ git commit -n -m "balabala"
```
