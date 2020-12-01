# 利用自带`qmp_shell`脚本自动化测试

1. ## 封装

   1. 利用`sys.path.append`增加搜索路径
   2. `os.path.join`是可以自动添加`/`路径分隔符的
   3. 我这里直接调用`QMPShell`类进行处理
      1. 用一个类`QMPFile`控制输入输出
      2. 在内部直接调用`QMPShell`的方法

2. ## 测试

3. ## 测试

