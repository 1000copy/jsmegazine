
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
# 章節設定
GitBook 使用一個 SUMMARY.md 檔案來定義書籍的目錄架構，也就是多層次章節的設定。SUMMARY.md 檔案同時也被用來製作書籍目錄（TOC - Tables Of Contents）。

SUMMARY.md 的格式只是簡單的連結列表，連結的「名稱」就是章節的「標題」，連結標的則是實際的內容「檔案」（包含路徑在內）。

章節的層級，就根據清單的層級進行定義。

GitBook 的這個設定方式相當彈性，在目錄中使用的「名稱」即使與實際檔案中的第一級大標題不同也無妨（但應注意不要造成讀者的困惑）；多層級可以將書分成「部」、「章」、「節」或是「小節」皆可，而且不會自動賦予編號或固定名稱，選擇自己想要的組織架構即可。

沒有在 SUMMARY.md 文件裏指定的檔案，GitBook 在轉製書籍時都不會使用，所以你可以自由撰寫草稿、參考文件等。

簡單的範例
	# Summary

	* [第一章](chapter1.md)
	* [第二章](chapter2.md)
	* [第三章](chapter3.md)
	以次章節將書分成 三大部 的範例
	# Summary

	* [第一部](part1/README.md)
	    * [寫作是美好的](part1/writing.md)
	    * [GitBook 也不錯](part1/gitbook.md)
	* [第二部](part2/README.md)
	    * [我們歡迎讀者回饋](part2/feedback_please.md)
	    * [對作者更好的工具](part2/better_tools.md)


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

