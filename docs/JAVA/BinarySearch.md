## 二分查找的思想
对有序集合通过类似分治的思想，每次都和区间的中间元素对比，每次折半缩小查询区间，知道找到元素或者区间大小为0，二分的时间复杂度为对数级O(logn)
## 给出有序集合和一个目标值，假定数组无重复元素，找出目标值索引位置，目标值如果不存在，返回按顺序应插入的索引位置
[leetcode链接](https://leetcode.com/problems/search-insert-position/)
```code
class Solution {
    public int searchInsert(int[] nums, int target) {
        int low =0;
        int high=nums.length-1;
        while(low<=high){
            int mid = low+(high-low)/2;
            if(nums[mid]==target){
                return mid;
            }else if(nums[mid]<target){
                low=mid+1;
            }else{
                high =mid-1;
            }    
        }
        return high+1;
        
    }
}
```


