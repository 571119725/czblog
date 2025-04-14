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

## 原地Hash

将数组转为hash表，用来计算一个数组中不存在的最小正整数。

不存在的正整数的范围为[1, nums.length]，一种做法是从1开始计算每个正整数是否存在与数组中，空间复杂度O(n)

另一种做法将数组视为哈希表，将1到nums.length之间的每个数字都交换到nums[i] - 1的数组位置，然后从头看数组下标处不符合规则的地方就是缺少的正整数位置

```java
public int firstMissingPositive(int[] nums) {
    int len = nums.length;
    for(int i = 0; i < len; i++) {
        while(nums[i] > 0 && nums[i] <= len && nums[nums[i] - 1] != nums[i]) {
            swap(nums, nums[i] - 1, i);
        }
    }
    for(int i = 0; i < len; i++) {
        if(nums[i] - 1 != i) return i + 1;
    }
    return len;
}
```

