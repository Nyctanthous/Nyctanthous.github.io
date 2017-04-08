

```python
import matplotlib.pyplot as plt

```


```python
test0List = [i for i in range(0, 10, 2)]
test1List = [i for i in range(0, 10, 4)]
test2List = [i for i in range(0, 10, 8)]
```


```python
plt.plot(test0List, [1] * len(test0List), 'ro')
plt.plot(test1List, [2] * len(test1List), 'bo')
plt.plot(test2List, [3] * len(test2List), 'go')
plt.axis([0, 10, 0, 5])
plt.show()
```


![png](2017-3-3-graphing_demo_files/2017-3-3-graphing_demo_2_0.png)


# MTF Lists

Given $n$ elements in any list, we propose there is exists an optimally ordered list $[x_0, x_1, x_2, x_3....]$
For a  given position $P_i$ in this list, $\forall i$, $P(x_i) \geq P(x_{i+1})$.

Let  $E(T_{opt})$ be the expected search time in the optimal list of these elements.

Let  $E(T_{MTF})$ be the expected search time in the MTF list of the same elements.

We may express the relation between $E(T_{opt})$ and $E(T_{MTF})$ as $E(T_{MTF}) < 2E(T_{opt})$, or the expected search time of a MTF list is, at worst, $2E(T_{opt})$


```python
t3List = [0,2,4,6,8,10]
t4List = [0,1,4,5,8,9]
t5List = [4,5,6,7,10,11]
t6List = [8,9,10,11]

plt.plot(t3List, [1] * len(t3List), 'ro')
plt.plot(t4List, [2] * len(t4List), 'bo')
plt.plot(t5List, [3] * len(t5List), 'go')
plt.plot(t6List, [4] * len(t6List), 'yo')
plt.axis([0, 10, 0, 5])
plt.show()
```


![png](2017-3-3-graphing_demo_files/2017-3-3-graphing_demo_4_0.png)



```python
import matplotlib.pyplot as plt

t0List = [0,2,4,6,8,10]
t1List = [0,1,4,5,8,9]
t2List = [4,5,6,7,10,11]
t3List = [8,9,10,11]

plt.plot(t0List, [1] * len(t0List), 'ro', \
         t1List, [2] * len(t1List), 'bo', \
         t2List, [3] * len(t2List), 'go', \
         t3List, [4] * len(t3List), 'yo')
plt.axis([0, 10, 0, 5])
plt.show()
```


![png](2017-3-3-graphing_demo_files/2017-3-3-graphing_demo_5_0.png)



```python
import pylab
import numpy as np

x = np.linspace(1,1000,1000) # 500 linearly spaced numbers
#a = 75*x
b = np.sqrt(x)
#c = 1.5**x
#d = x**2 + np.log2(x)
e = x**1.5


f = x * np.log10(x)
g = x * np.log2(x)

# compose plot
pylab.plot(x,b, 'go')
pylab.plot(x,f, 'ro')
pylab.plot(x,g)
pylab.show() # show the plot
```


![png](2017-3-3-graphing_demo_files/2017-3-3-graphing_demo_6_0.png)



```python

```
