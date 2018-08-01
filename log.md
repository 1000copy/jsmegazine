
# 制作gitbook电子书的过程

## 前条件

首先你必須已經安裝好 Node、NPM 以及 Calibre。

# make book
	
	#安装gitbook
	npm install gitbook-cli -g
	# 生成PDF文件
	gitbook pdf . 1.pdf
	open 1.pdf

# 添加封面

無需廢言，一個清晰漂亮的書籍封面是必要的。

在 GitBook 書籍專案根目錄下擺一個 cover.jpg （大書封）以及一個 cover_small.jpg （小書封）。

注意：只接受 JPEG 圖片格式。其他什么PNG都不可以的。

# 解决问题

## 安装gitbook-cli报错

安装执行命令：

	 npm install gitbook-cli -g

总是报错：

	npm ERR! path /usr/local/lib/node_modules/gitbook-cli/node_modules/npmi/node_modules/npm/node_modules/ansistyles/npm-shrinkwrap.json
	npm ERR! code ENOTDIR
	npm ERR! errno -20
	npm ERR! syscall open
	
结果我删除了ansistyles文件，并创建此目录，就对了。

	rm /usr/local/lib/node_modules/gitbook-cli/node_modules/npmi/node_modules/npm/node_modules/ansistyles
	mkdir /usr/local/lib/node_modules/gitbook-cli/node_modules/npmi/node_modules/npm/node_modules/ansistyles/

太好笑了。
