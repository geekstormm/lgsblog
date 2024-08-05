# Leetcode 2216

直接模拟

```c++
class Solution {
public:
    int minDeletion(vector<int>& nums) {
       int n = nums.size();
       int cnt = 0;
       for (int i = 0; i < n - 1; ++i) {
            if ((i - cnt) % 2 == 0 && nums[i] == nums[i + 1]) {
                cnt++;
            }
       }

       if ((n - cnt) % 2 == 1) return cnt + 1;
       return cnt;
    }
};
```



