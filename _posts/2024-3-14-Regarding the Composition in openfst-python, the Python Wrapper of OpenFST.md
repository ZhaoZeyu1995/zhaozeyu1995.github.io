---
title:  "Regarding the Composition in openfst-python, the Python Wrapper of OpenFST"
mathjax: true
---

This post is about the composition operate in `openfst-python`.
It is demanded by the (official documentation)[https://www.openfst.org/twiki/bin/view/FST/ComposeDoc] that before composition,
the output labels of the first FST or the input labels of the second FST must be sorted.
However, I find that, it seems not to be the case in the Python wrapper, `openfst-python`.

Here is a simple example to demonstrate it.

```python
import openfst_python as fst
```


```python
compiler = fst.Compiler()
print("0 1 1 8 0", file=compiler)
print("0 1 2 7 0", file=compiler)
print("0 1 3 6 0", file=compiler)
print("0 1 4 5 0", file=compiler)
print("0 1 5 9 0", file=compiler)
print("1 0", file=compiler)
f = compiler.compile()
```


```python
f
```




    
![svg](/_posts/2024-3-14/output_2_0.svg)
    




```python
compiler = fst.Compiler()
print("0 1 5 4 0", file=compiler)
print("0 1 6 3 0", file=compiler)
print("0 1 7 2 0", file=compiler)
print("0 1 8 1 0", file=compiler)
print("0 1 9 5 0", file=compiler)
print("1 0", file=compiler)
g = compiler.compile()
```


```python
g
```
    
![svg](/_posts/2024-3-14/output_4_0.svg)

 There is something worth noting in `f` and `g`.
First, if the input labels `f` are sorted, then the output labels of `f` cannot be in sorted state.
This is the same for `g`.
Second, the composition result of `f` and `g` is simple as they are the inverse version of each other. 
   




```python
fi = f.arcsort(sort_type="ilabel")
fo = f.arcsort(sort_type="olabel")
gi = g.arcsort(sort_type="ilabel")
go = g.arcsort(sort_type="olabel")
```


```python
fi
```




    
![svg](/_posts/2024-3-14/output_6_0.svg)
    




```python
fo
```




    
![svg](/_posts/2024-3-14/output_7_0.svg)
    




```python
gi
```




    
![svg](/_posts/2024-3-14/output_8_0.svg)
    




```python
go
```




    
![svg](/_posts/2024-3-14/output_9_0.svg)
    




```python
h = fst.compose(f, g)
hio = fst.compose(fi, go)
hoo = fst.compose(fo, go)
hii = fst.compose(fi, gi)
hoi = fst.compose(fo, gi)
```


```python
h
```




    
![svg](/_posts/2024-3-14/output_11_0.svg)
    




```python
hio
```




    
![svg](/_posts/2024-3-14/output_12_0.svg)
    




```python
hoi
```




    
![svg](/_posts/2024-3-14/output_13_0.svg)
    




```python
hii
```




    
![svg](/_posts/2024-3-14/output_14_0.svg)
    




```python
hoo
```




    
![svg](/_posts/2024-3-14/output_15_0.svg)
    




```python

```

We can see that no matter how we do the `arcsort` for `f` and `g`, or even do not sort them at all, we can always do the composition without any error.
This is different from what the official documentation described.

Here is the [notebook file](/_posts/2024-3-14/debug.ipynb), if you'd like to play around.