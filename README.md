# react-cropper antDesign
使用react-cropper和antDesign的Upload组件实现的图片裁剪插件
# 一.首先安装react-cropper插件
npm install --save react-cropper
执行该命令以后，下载react-cropper依赖信息自动更新到package.json中
在使用该插件的代码中需要进行引入

import "cropperjs/dist/cropper.css"
import Cropper from 'react-cropper'

# 二.主要的实现的功能
使用antdesign的upload组件进行上传图片，并调用react-cropper组件实现固定长和宽的截图，将截图的结果以FILE格式传给后台。实现效果如下。
![image](https://github.com/liusiasi/react-cropper/raw/master/picture/first.png)
可以上传多张图片。

# 三.主要的思路
首先构建一个CropperUpload组件，该组件接收截图框的长，宽，Upload样式模式，初始图片列表四个参数。该组件主要实现上传可截图的图片的功能，在Upload组件的beforeUpload中用FileReader读取要上传的文件的base64码，并返回false阻止Upload组件自动上传，将文件码赋值给cropper组件，用cropper进行截图，截图之后，将cropper返回的base64码转换成File格式传向后台。
