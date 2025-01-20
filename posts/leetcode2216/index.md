# Leetcode2216


&lt;!--more--&gt;


## Leetcode 2216

直接模拟

```c&#43;&#43;
class Solution {
public:
    int minDeletion(vector&lt;int&gt;&amp; nums) {
       int n = nums.size();
       int cnt = 0;
       for (int i = 0; i &lt; n - 1; &#43;&#43;i) {
            if ((i - cnt) % 2 == 0 &amp;&amp; nums[i] == nums[i &#43; 1]) {
                cnt&#43;&#43;;
            }
       }

       if ((n - cnt) % 2 == 1) return cnt &#43; 1;
       return cnt;
    }
};
```

---

> 作者: [LGS](https://github.com/geekstormm)  
> URL: https://geekstormm.github.io/posts/leetcode2216/  

