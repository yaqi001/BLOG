# 经典

~~~ python
>>> numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
>>> numbers[0:-1]
[1, 2, 3, 4, 5, 6, 7, 8, 9]
>>> numbers[-2:]
[9, 10]
>>> numbers[0::3]
[1, 4, 7, 10]
~~~

~~~ python
>>> zip([1,2,3,4,5,6], [2,3,4,588])
[(1, 2), (2, 3), (3, 4), (4, 588)]
>>> zip(range(2), range(5, 7), range(1,99))
~~~

~~~ python
>>> zip(range(4), range(2,6))
[(0, 2), (1, 3), (2, 4), (3, 5)]
~~~