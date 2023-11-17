---
layout: post
title:  "Modeling a translation of a point with a snippet of Haskell"
date:   2023-11-16 6:20:20 -0800
categories: haskell programming
---

Today I realized that Haskell is a great way to model translations of a point on the coordinate plane.

In math class, an example was given in today's lesson that for any point `p`, a translation of `x, y` and another of `a, b` was the same as a single translation of `x+a, y+b`. Looks familiar? It has a striking resemblance to function composition in Haskell, as a coordinate plane can be represented as a undirected graph.

So, I thought, why not give it a try?

```hs
main = do
  print $ transform 2 3 (Point 1 1)
  
data Point = Point Int Int deriving (Eq, Show) -- x and y coordinates

transform :: Int -> Int -> Point -> Point -- tranform the x and y coordinates of a point
transform dx dy (Point x y) = Point (dx + x) (dy + y)
```

```hs
Point 4 5
```

Great! Transforming the coordinates of a point is easy enough. Now, all we need to do is chain multiple translations with the help of [function currying](https://wiki.haskell.org/Currying) and [function composition](https://wiki.haskell.org/Function_composition).

```hs
main = do
  print $ (transform 1 2) . (transform 2 3) $ (Point 1 1)
```

```hs
Point 5 7
```

We can now use this to assert that the example in my math class was right:

```hs
main = do
  print $ ((transform 1 2) . (transform 2 3) $ (Point 1 1)) == transform 3 5 (Point 1 1)
```

```hs
True
```

Very cool!
