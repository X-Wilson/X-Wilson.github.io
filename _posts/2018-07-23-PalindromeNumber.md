---
title: PalindromeNumber
date: 2018-07-23 21:50:00
tags: 
- Leetcode
excerpt_separator: <!--more-->
---
该问题的[Leetcode地址](https://leetcode.com/problems/palindrome-number/description/)
<!--more-->
## 问题描述：
>Determine whether an integer is a palindrome. An integer is a palindrome when it reads the same backward as forward.  
>**Example 1:**
>```
Input: 121
Output: true
>```
>**Example 2:**
>```
>Input: -121
Output: false
Explanation: From left to right, it reads -121. From right to left, it becomes 121-. Therefore it is not a palindrome.
>```
>**Example 3:**
>```
>Input: 10
Output: false
Explanation: Reads 01 from right to left. Therefore it is not a palindrome.
>```

## Solution1:
这个问题和之前做过的[这题](https://leetcode.com/problems/reverse-integer/description/)有点类似，不过当时并没有记录下做法，但是在思路上可以借鉴反向输出的做法，也就是用`取余法`来对给出的整数由低位到高位的遍历输出，因为所需要输出的位数不多，所以可以直接使用`ArrayList`来进行操作，而不使用`LinkedList`，可以参考下这一篇记录——先占个坑，有空来填完[链接]()  
```java
class Solution {
    public boolean isPalindrome(int x) {
    	boolean flag = false;
        List<Integer> listNums1 = new ArrayList<>();//用于接收取余的数组
        List<Integer> listNums2 = new ArrayList<>();//比较数组
        //特殊情况
        if(x==0)return true;
        if(x<0) return false;
        
        while(x!=0){
           listNums1.add((Integer)(x % 10));
           x /= 10;
        }
        for(int i=listNums1.size()-1;i>=0;i--){
        	listNums2.add(listNums1.get(i));
        }
        if(listNums1.equals(listNums2))
        	flag = true;
        return flag;
    }
}
```
## Solution2:
做完上面那个解法，在API里无意间看到了`Collection`这个工具类，想着可不可以用里面的reverse()方法来试一下，将得到的逆向数组做一次反向，再做比较，于是乎有了下面的解法。  
这种方法需要使用到Java的`CopyOnWriteArrayList`这个类。我在本地调试时没有问题，不过放到Leetcode上时，编译会报错，原因是找不到CopyOnWriteArrayList这个类，不知道是怎么回事。  
（更新一下，刚刚review了一下post上去的代码，是没有加上下面的`implort`语句的，我还以为Leetcode会自动导入，可能官方只是会自动导入Java的一些基本包而已，像`java.lang`，`java.util`，而超过2层的包都需要自己导入）。

```java
import java.util.concurrent.CopyOnWriteArrayList;

class Solution {
    public boolean isPalindrome(int x) {
    	boolean flag = false;
        List<Integer> listNums1 = new ArrayList<>();
        if(x==0)return true;
        if(x<0) return false;
        while(x!=0){
           listNums1.add((Integer)(x % 10));
           x /= 10;
        }
        CopyOnWriteArrayList<Integer> listNums2 = new CopyOnWriteArrayList<>(listNums1);
        Collections.reverse(listNums2);
        if(listNums1.equals(listNums2))
        	flag = true;
        return flag;
    }
}
```
## Runtime比较
我把这两种解法都post上去，不过前者所花费的时间会比较少  

|Solution|Question|Status|Runtime|Language|
|:--:|:--:|:--:|:--:|:--:|:--:|
|Solution1|Palindrome Number|Accept|115 ms|Java|
|Solution2|Palindrome Number|Accept|137 ms|Java|

刚好不犯困，就索性来看看`CopyOnWriteArrayList`这个类是什么回事。  
然后发现在`CopyOnWriteArrayList`的源代码的这个构造方法中，使用的是Arrays这个工具类对传进去的泛型对象数组进行转换。  
```java
/**
     * Creates a list containing the elements of the specified
     * collection, in the order they are returned by the collection's
     * iterator.
     *
     * @param c the collection of initially held elements
     * @throws NullPointerException if the specified collection is null
     */
public CopyOnWriteArrayList(Collection<? extends E> c) {
        Object[] elements;
        if (c.getClass() == CopyOnWriteArrayList.class)//这个是Java的反射机制（即从实例对象反推出Java类）
            elements = ((CopyOnWriteArrayList<?>)c).getArray();
        else {
            elements = c.toArray();
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elements.getClass() != Object[].class)
                elements = Arrays.copyOf(elements, elements.length, Object[].class);//转换
        }
        setArray(elements);
    }
```
所以这一步转过来转过去，只是为了copy一个对象，而且之后还是需要执行reverse()操作。  
即使是在Collections.reverse()这个方法内部，执行的也是for循环的操作，和`Solution1`是相似的操作，所以Runtime的性能损失在了上面那样一个操作当中了。
```java
/**
     * Reverses the order of the elements in the specified list.<p>
     *
     * This method runs in linear time.
     *
     * @param  list the list whose elements are to be reversed.
     * @throws UnsupportedOperationException if the specified list or
     *         its list-iterator does not support the <tt>set</tt> operation.
     */
    @SuppressWarnings({"rawtypes", "unchecked"})
    public static void reverse(List<?> list) {
        int size = list.size();
        if (size < REVERSE_THRESHOLD || list instanceof RandomAccess) {
            for (int i=0, mid=size>>1, j=size-1; i<mid; i++, j--)
                swap(list, i, j);
        } else {
            // instead of using a raw type here, it's possible to capture
            // the wildcard but it will require a call to a supplementary
            // private method
            ListIterator fwd = list.listIterator();
            ListIterator rev = list.listIterator(size);
            for (int i=0, mid=list.size()>>1; i<mid; i++) {
                Object tmp = fwd.next();
                fwd.set(rev.previous());
                rev.set(tmp);
            }
        }
    }
```
