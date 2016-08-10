---
layout: post
title:  "Commit message 格式化"
date:   2015-12-27
comments: true
categories: javascript
---

* content
{:toc}

###validate-commit-msg 

用于检查 Node 项目的 Commit message 是否符合格式。

它的安装是手动的。首先，拷贝下面这个JS文件，放入你的代码库。文件名可以取为[validate-commit-msg.js](https://github.com/kentcdodds/validate-commit-msg/blob/master/index.js)。

接着，把这个脚本加入 Git 的 hook。下面是在package.json里面使用 ghooks（没有该模块，需安装），把这个脚本加为commit-msg时运行。

	"config": {
		"ghooks": {
		  "commit-msg": "./validate-commit-msg.js"
		}
	}
	
然后，每次git commit的时候，这个脚本就会自动检查 Commit message 是否合格。如果不合格，就会报错。

	$ git add -A 
	$ git commit -m "edit markdown" 
	INVALID COMMIT MSG: does not match "<type>(<scope>): <subject>" ! was: edit markdown

### 格式
* feat：新功能（feature）
* fix：修补bug
* docs：文档（documentation）
* style： 格式（不影响代码运行的变动）
* refactor：重构（即不是新增功能，也不是修改bug的代码变动）
* test：增加测试
* chore：构建过程或辅助工具的变动
	
	
type(scope): subject
	
	$chore(package): update
