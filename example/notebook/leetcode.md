## 单调队列

计算固定大小滑动窗口内的最大值（最小值）

使用一个队列来管理最值，如果新添加的元素大小比队尾的数字小便可以正常添加，如果比队尾的数字大，则当前滑动窗口内的最值调整，将队尾数字移除，直到队列无元素或遇到比当前元素大的，队列中的元素体现递减趋势。

窗口滑动时，需要比较队头的元素和滑动窗口要删除的元素是否一样，相同的话直接删除，表明这个最值过期了，但是不会影响其他元素在队列中存储的最值。

```java
class MyDeque {
    Deque<Integer> queue = new LinkedList<>();
    public void add(int cur) {
        while(!queue.isEmpty() && queue.getLast() < cur) {
            queue.removeLast();
        }
        queue.addLast(cur);
    }
    public void poll(int cur) {
        if(!queue.isEmpty() && cur == queue.getFirst()) {
            queue.removeFirst();
        }
    }
    public int peek() {
        return queue.getFirst();
    }
}
```

## 前缀和

计算数组中子数组和为k的数量

子数组i ~ j 的和可以表示为sum(nums[j]) - sum(sums[i])，并判断是否等于k，所以可以使用hashmap存储sum(nums[i])的出现次数，然后通过遍历数组查看sum(nuts[j]) - k的key在map中是否存在，然后更新相关出现次数。

```java
for(int i = 0; i < nums.length; i++) {
    count += nums[i];
    if(map.containsKey(count - k)) {
        result += map.get(count - k);
    }
    map.put(count, map.getOrDefault(count, 0) + 1);
}
```

