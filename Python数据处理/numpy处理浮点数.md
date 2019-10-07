在处理论文的数据时，因为浮点数存在精度问题，本来该是0的数，结果为6.66133814775e-16，我们如何处理这种浮点数呢？

可以使用numpy的finfo创建指定精度范围的浮点数

| 数据类型 | 实际大小 |
| -------- | -------- |
| float16  | 16bit    |
| float32  | 32bit    |
| float64  | 64bit    |
| float96  | 96bit    |
| float128 | 128bit   |

```python
>>> np.finfo(np.float32).eps
1.1920929e-07
>>> np.finfo(np.float64).eps
2.2204460492503131e-16

>>> np.float32(1e-8) + np.float32(1) == 1
True
>>> np.float64(1e-8) + np.float64(1) == 1
False
```

```python
 val if abs(val) > 10 * np.finfo(np.float).eps else 0
```

## 参考资料

[numpy如何优雅地将6.6e-16的小数显示为0？](https://segmentfault.com/q/1010000011791590)

[**彻底剖析numpy的数据类型**](https://blog.csdn.net/gg_18826075157/article/details/78609532)

