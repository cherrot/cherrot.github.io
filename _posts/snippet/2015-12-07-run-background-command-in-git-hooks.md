---
layout: post
title: "在git hook中执行后台进程"
date: 2015-12-07 21:39:00
categories: snippet
tags: git
keywords: 
description: "howto: run background (time consuming) command in git hooks like post-receive"

---
试过N个方案，最终可行的是结合重定向与`disown`的方案(注释掉的都是不可行的方法)：

```
COMMAND </dev/null &>/dev/null & disown
#exec COMMAND >&- 2>&- &
#COMMAND |at now
#nohup COMMAND 2>&1 >/dev/null &
```

附上我在生产环境中使用的前端代码自动部署脚本
(`/path/to/my/repo.git/hooks/post-receive`):

```
#!/bin/bash
#to enable code rollback (by `git push -f`):
#git config receive.denynonfastforwards false

SRC_DIR=/Path/to/checkout/your/original/code
ROOT_DIR=/Path/to/your/deployment/target
 
GIT_WORK_TREE=$SRC_DIR git checkout -f
# if set GIT_WORK_TREE twice, file modification time would be overwritten
# GIT_WORK_TREE=$ROOT_DIR git checkout -f

rm -rf ${ROOT_DIR}.old
cp -rp $SRC_DIR ${ROOT_DIR}.new
mv $ROOT_DIR ${ROOT_DIR}.old
mv ${ROOT_DIR}.new $ROOT_DIR
 
config_path=$ROOT_DIR/static/js/lib/require/require-config.js
 
for file in $(grep -oP "(?<=[\'\"])([^\'^\"]+\.js)(?=[^\'^\"]*[\'\"])" $config_path); do
    if [ -f $SRC_DIR/static/js/$file ]; then
        last_modify=$(date +%m%d%H%M -r $SRC_DIR/static/js/$file)
        sed -i "s@$file[^\'^\"]*@${file}?_=$last_modify@g" $config_path
    fi
done

find $ROOT_DIR/static/js -type f -name "*.js" -exec uglifyjs {} -o {} \; </dev/null &>/dev/null & disown

#exec find . -type f -name "*.js" -exec uglifyjs {} -o {} \; >&- 2>&- &
#find . -type f -name "*.js" -exec uglifyjs {} -o {} \; |at now
#nohup find . -type f -name "*.js" -exec uglifyjs {} -o {} \; 2>&1 >/dev/null &
exit
```
