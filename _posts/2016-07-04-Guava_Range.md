---
layout: post
title: Guava Range
categories: [blog]
tags: [Guava]
description: 
---

## Guava Range

区间实例可以由[Range](http://docs.guava-libraries.googlecode.com/git/javadoc/com/google/common/collect/Range.html)类的静态方法获取

(a..b)|open(C,C)
[a..b]|closed(C,C)
[a..b)|closedOpen(C,C)
(a..b]|openClosed(C,C)
(a..+∞)|greaterThan(C)
[a..+∞)|atLeast(C)
(-∞..b)|lessThan(C)
(-∞..b]|atMost(C)
(-∞..+∞)|all()

### Contains方法

Range的基本运算是它的[contains(C)](http://docs.guava-libraries.googlecode.com/git/javadoc/com/google/common/collect/Range.html#contains(C)) 方法，和你期望的一样，它用来区间判断是否包含某个值

```
    @Test
    public void testBase() {
        Range<Integer> range3 = Range.closedOpen(0,9);
        printRange(range3);
        printRange(Range.closed(0,9));
        printRange(Range.openClosed(0,9));
        printRange(Range.open(0,9));
        printRange(Range.closedOpen(0,9));

        Assert.assertEquals(Range.closed(0,9).contains(8),true);
        Assert.assertEquals(Range.closed(0,9).contains(10),false);
        Assert.assertEquals(Range.closed(0,9).containsAll(Ints.asList(1,2,3)),true);

    }
```



### 包含[enclose]

```java
/**包含
 * Range.enclose(Range) 如果区间被外部包含,则返回true,否则返回false
 */
@Test
public void testEnclose(){
    Assert.assertEquals(Range.closed(0,9).encloses(Range.closed(2,5)),true);
    Assert.assertEquals(Range.closed(2,5).encloses(Range.closed(0,9)),false);
}
```

### 交集[intersection]

```java
    /**交集
     * Range.intersection() 返回两个区间的交集：既包含于第一个区间，又包含于另一个区间的最大区间。
     * 当且仅当两个区间是相连的，它们才有交集。如果两个区间没有交集，该方法将抛出IllegalArgumentException。
     */
    @Test
    public void testIntersection() {
        printRange(Range.closed(3,5).intersection(Range.open(5,10)));
        printRange(Range.closed(3,5).intersection(Range.closed(5,10)));
    }
```

### 跨区间[span]

```java


/**跨区间
 *Range.span(Range range) 同时包括两个区间的最小区间”，如果两个区间相连，那就是它们的并集
 */
@Test
public void testSpan() {
    printRange(Range.closed(3,5).span(Range.open(5,10)));
    printRange(Range.closed(3,5).span(Range.open(10,15)));
}
```

### 相连[isConnected]



```java
    /**相连
     * Range.isConnected(Range)测试这些区间是否是连续
     * 具体来说，isConnected测试是否有区间同时包含于这两个区间，
     * 这等同于”两个区间的并集是连续集合的形式”（空区间的特殊情况除外）。
     */
    @Test
    public void testIsConnected() {
        Assert.assertEquals(Range.closed(3, 5).isConnected(Range.open(5, 10)),true);
        Assert.assertEquals(Range.closed(0, 9).isConnected(Range.closed(3, 4)),true);
        Assert.assertEquals(Range.closed(0, 5).isConnected(Range.closed(3, 9)),true);
        Assert.assertEquals(Range.open(3, 5).isConnected(Range.open(5, 10)),false);
        Assert.assertEquals(Range.closed(1, 5).isConnected(Range.closed(6, 10)),false);
    }
```

