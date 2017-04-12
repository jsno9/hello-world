---
layout: post
title: "linux xls文件解析"
description:
category: linux
tags: [linux]
mathjax: 
chart:
comments: false
---

###1.为什么要解析xls
做项目的过程中需要与其他部门沟通，会push其他部门填表，只要定好表的格式，再结合我们xls解析代码，即可自动化完成xls解析，拿到我们需要的数据，节省人力，减少出错

###2.linux c实现xls文件解析
1. 使用第三方库libxl(http://www.libxl.com/)

2. 具体举例请看https://github.com/jsno9/public/tree/master/tools/xlsanalysis

截取其中解析xls的代码

	{		
		BookHandle book = xlCreateBook();	
		if(book) 
		{
			printf("book ok\n");
			if(xlBookLoad(book, "thermal1.xls"))	//打开需要解析的xls文件 	
			{
				printf("xlBookLoad ok\n");
				SheetHandle sheet = xlBookGetSheet(book, 1);	//xls文件一般有过个sheet，选择需要解析的sheet
				const char* s = xlSheetReadStr(sheet, x, y, 0);	//读取xls文件指定sheet中指定行列中数据
			}
		}
	}

+ 这是一个最最简单的xls的读取

###3.python实现xlx文件解析
1. 依然要使用第三方的python库，xlrd，下载xlrd并且安装

2. 具体使用用例请看https://github.com/jsno9/public/tree/master/python/xlsanalysis
