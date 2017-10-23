# diff trailing CR

乔俊峰 Junfeng Qiao， 2017-10-23

linux下以LF作为换行符，windows下是CR/LF。windows下写的python代码在linux下运行前要先用vi的`:set fileformat=unix`或`:set ff=unix`或者`dos2unix`来转换成linux的文本格式。

有时候分别在两个平台改过代码并保存成了不同的文件，这时候需要`diff`来看看区别。但是`diff`默认会将行尾LF和CR/LF也视为不同，此时只要使用`diff --strip-trailing-cr file1 file2`参数就可忽略换行符的差异。