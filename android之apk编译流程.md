apk编译流程
==========

1. 打包资源文件,生成R.java文件（使用工具AAPT）
2. 处理AIDL文件，生成java代码
3. 编译java文件，生成对应的.class文件（java compiler）
4. .class文件转换成dex文件(dex)
5. 打包成没有签名的apk(使用工具apkbuilder)
6. 使用签名工具给apk签名(使用工具jarsigner)
7. 对签名后的文件进行对齐处理，不进行对齐处理不能发布到Google Market（使用工具zipalign）
