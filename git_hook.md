## 使用git hook验证php语法错误

#### 效果如下:(在git commit 前对提交文件进行php -l 语法检测)

![效果图](http://airzhe.github.io/images/md/git_hook/1.png)

#### 实现方法

	vim  pre-commit

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

然后执行以下操作

	cp pre-commit .git/hooks/pre-commit; 
	chmod +x .git/hooks/pre-commit;

这样就可以了.会再你commit 之前检查php语法,如果不通过,会报错,并阻止提交

#### 参考资料

- [Git pre-commit хук в Windows](http://plutov.by/post/git_pre_commit_windows)
- [An Introduction to Git Hooks](https://www.sitepoint.com/introduction-git-hooks/)
