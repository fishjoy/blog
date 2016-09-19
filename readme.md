# Albert Cheng's blog: 记录一些学习和填过的坑
###spark streaming应用一个越跑越慢的bug
在写一个spark streaming应用时，遇到了一个奇怪的bug，7*24小时运行之后程序慢了一个数量级，而且在持续慢。记录解bug的过程。