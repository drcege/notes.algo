[toc]


# 介绍

`autograd` 包提供了所有操作在张量上的自动微分。反向传播过程会根据网络的前向传播自动定义，这意味着可以在每次迭代中动态改变网络的结构。


# 张量 Tensor

* 如果将 `torch.Tensor` 的属性 `.requires_grad` 设置为 `True`，则它会记录它上面的所有 operations
* 完成前向计算后，调用 `.backward()` 方法，所有张量的梯度都会自动计算，并保存在该张量的 `.grad` 属性中
* 要在某个张量处停止跟踪历史记录，调用其 `.detach()` 方法将其从计算历史中移除，那么之后的所有计算都不会被追踪
* 要完全停止跟踪历史记录，将代码块置于 `with torch.no_grad():` 中。这样在评估模型时可以节省大量内存，因为不需要再对所有张量计算梯度！


# 函数 Function

* Tensor 和 Function 相互连接构成无环图，编码了完整的计算历史
* 每个 Tensor 有个 `.grad_fn` 属性，指向创建了这个张量的 Function（由用户直接创建的 Tensor 其 grad_fn 为 None）
* 在张量上调用 `.backward()` 方法可以计算 grad_fn 的导数。如果 Tensor 是一个标量，调用 `.backward()` 不需要参数，否则，需要传入一个形状匹配的 `gradient` 参数


# 计算图

设置 Tensor 的 `requires_grad=True` 以开始追踪计算历史：

```python
import torch

x = torch.ones(2, 2, requires_grad=True)
print(x)

输出：
tensor([[1., 1.],
        [1., 1.]], requires_grad=True)
```

在张量上进行操作，可以看到自动记录了其 `grad_fn`：

```python
y = x + 2
print(y)
print(y.grad_fn)

输出：
tensor([[3., 3.],
        [3., 3.]], grad_fn=<AddBackward0>)
<AddBackward0 object at 0x7f6de4b1eb70>
```

用 `.requires_grad_(requires_grad=True)` 方法可以原地控制该 Tensor 是否需要计算梯度：

```python
a = torch.randn(2, 2)
a = ((a * 3) / (a - 1))
print(a.requires_grad)

a.requires_grad_(True)
print(a.requires_grad)

b = (a * a).sum()
print(b.grad_fn)

输出：
False
True
<SumBackward0 object at 0x7f6de4b295f8>
```


# 梯度 Gradient

## 标量输出的反向传播

如果函数输出是标量，直接调用 Tensor 的 `.backward()` 等价于 `.backward(torch.tensor(1.))`。如计算 $\frac{\partial out}{\partial x}$:

```python
out.backward()
print(x.grad)

输出：
tensor([[4.5000, 4.5000],
        [4.5000, 4.5000]])
```


### 手动计算验证

用 o 表示 out, 有：$o = \frac{1}{4}\sum_i z_i,\, z_i = 3(x_i + 2)^2,\, z_i\vert_{x_i = 1} = 27$

则 $\frac{\partial o}{\partial x_i} = \frac{3}{2}(x_i + 2),\,\frac{\partial o}{\partial x_i}\vert_{x_i = 1} = 4.5$



## 非标量输出的反向传播

如果 $\vec{y} = f(\vec{x})$ 的输出是向量，那么 $\vec{y}$ 关于 $\vec{x}$ 的梯度是一个 Jacobian 矩阵：
$$
J = 
\begin{bmatrix}
   \frac{\partial y_1}{\partial x_1} & \cdots & \frac{\partial y_1}{\partial x_n} \\
   \vdots & \ddots & \vdots \\
   \frac{\partial y_m}{\partial x_1} & \cdots & \frac{\partial y_m}{\partial x_n}
\end{bmatrix}
$$


而 `torch.autograd` 引擎总是计算 vector-Jacobian 乘积，即假设有一个标量输出函数 $l = g(\vec{y})$，而 $v$ 是 $l$ 关于 $\vec{y}$ 的梯度：

$$
v = (\frac{\partial l}{\partial y_1}, \cdots, \frac{\partial l}{\partial y_m})^T,
$$

根据链式法则，vector-Jacobian 乘积 $J^T \cdot v$ 就是在计算 $l$ 关于 $\vec{x}$ 的梯度：

$$
\frac{\partial l}{\partial \vec{x}} = \frac{\partial l}{\partial \vec{y}} \frac{\partial \vec{y}}{\partial \vec{x}} = J^T \cdot v = \begin{bmatrix}
   \frac{\partial y_1}{\partial x_1} & \cdots & \frac{\partial y_m}{\partial x_1} \\
   \vdots & \ddots & \vdots \\
   \frac{\partial y_1}{\partial x_n} & \cdots & \frac{\partial y_m}{\partial x_n}
\end{bmatrix} \begin{bmatrix}
   \frac{\partial l}{\partial y_1}\\
   \vdots \\
   \frac{\partial l}{\partial y_m}
\end{bmatrix} = \begin{bmatrix}
   \frac{\partial l}{\partial x_1}\\
   \vdots \\
   \frac{\partial l}{\partial x_n}
\end{bmatrix}
$$

这就是为什么在非标量 Tensor 上调用  `.backward()` 时需要传入 gradient 参数，而且这也方便我们自由注入梯度作为权重。

### 代码示例

1. 定义计算图：

```python
x = torch.randn(3, requires_grad=True)

y = x * 2
while y.data.norm() < 1000:
    y = y * 2

print(y)

输出：
tensor([ 995.2585, 1440.6962,  307.3500], grad_fn=<MulBackward0>)
```

2. 由于输出 $\vec{y}$ 是向量，要计算 $\vec{y}$ 对 $\vec{x}$ 的梯度，需要注入假想的 $l$ 对 $\vec{y}$ 的梯度：

```python
y.backward(torch.tensor([0.1, 1.0, 0.0001], dtype=torch.float)）
print(x.grad)

输出：
tensor([1.0240e+02, 1.0240e+03, 1.0240e-01])
```

3. 停止梯度计算：

```python
print(x.requires_grad)
print((x ** 2).requires_grad)

with torch.no_grad():
    print((x ** 2).requires_grad)
	
输出：
True
True
False
```

或者用 `.detach()` 将 Tensor 从计算历史中拿出，得到一个相同值但不再需要梯度的张量：

```
print(x.requires_grad)
y = x.detach()
print(y.requires_grad)
print(x.eq(y).all())

输出：
True
False
tensor(True)
```



