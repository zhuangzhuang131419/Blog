---
title: LeetCode那些事
date: 2020-06-29 13:47:40
tags:
---

# Q146 LRU Cache
Design and implement a data structure for Least Recently Used (LRU) cache. It should support the following operations: get and put.

get(key) - Get the value (will always be positive) of the key if the key exists in the cache, otherwise return -1.
put(key, value) - Set or insert the value if the key is not already present. When the cache reached its capacity, it should invalidate the least recently used item before inserting a new item.

The cache is initialized with a positive capacity.

## 算法实现
### LinkedHashMap
#### LinkedHashMap vs. HashMap
大多数情况下，只要不涉及线程安全问题，Map基本都可以使用HashMap，不过HashMap有一个问题，就是迭代HashMap的顺序并不是HashMap放置的顺序，也就是无序。HashMap的这一缺点往往会带来困扰，因为有些场景，我们期待一个**有序**的Map.

* LinkedHashMap = LinkedList + HashMap

* accessOrder   
    * false： 基于插入顺序    
    * true：  基于访问顺序 

#### 参考文献
https://juejin.im/post/5a4b433b6fb9a0451705916f

### 具体实现

```java
public class LRUCache {
    private int capacity;
    private LinkedHashMap<Integer, Integer> map;

    public LRUCache(int capacity) {
        this.capacity = capacity;
        map = new LinkedHashMap<>(capacity, 1, true);
    }

    public int get(int key) {
        if (!map.containsKey(key)) {
            return -1;
        }
        return map.get(key);
    }

    public void put(int key, int value) {
        if (!map.containsKey(key)) {
            if (map.size() >= capacity) {
                for (Map.Entry<Integer, Integer> entry : map.entrySet()) {
                    map.remove(entry.getKey());
                    break;
                }
            }
        }
        map.put(key, value);
    }
}

```
