## 使用git hook验证php语法错误

效果如下:(在git commit 前对提交文件进行php -l 语法检测)

![效果图](http://airzhe.github.io/images/md/git_hook/1.png)

### 实现方法

`vim  pre-commit`

代码如下

	#!/bin/sh
	php_files=$(git status --short | grep -E '^(A|M)' | awk '{ print $2 }' | grep -E '\.php$')
	for file in $php_files; do
		php_out=$(php -l "$file" 2>&1)
		if ! [ $? -eq 0 ]; then
			echo "Syntax error with ${file}:"
			echo "$php_out" | grep -E '^Parse error'
			exit 1
		fi
	done



