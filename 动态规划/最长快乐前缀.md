# 最长快乐前缀
[题目链接](https://leetcode-cn.com/problems/longest-happy-prefix/)

## 解决方法
`Rabin-Karp指纹字符串`

### Rabin-Karp指纹字符串

#### 思路

* 制定`R`进制，其中`R` = 31
* 对于字符串`S`，从`i`下标到`j`下标的字符串哈希值为`H(i, j)`。那么：
    * `H(i, j + 1)` = `H(i, j)` * `R` + `H(j + 1, j + 1)`
    * `H(i - 1, j)` = `H(i, j)` + `H(i - 1, i - 1)` * `R`^(j - i + 1)
* 从字符串头部和尾部，遍历获取前缀对比哈希值，找到哈希值相等的最大长度

#### 代码
```java
public class Solution {

    public String longestPrefix(String s) {
        int length = 0;
        int a = 31;
        
        for (int i = 0, j = s.length() - 1, k = 0, n = 0, m = 1; i < s.length() - 1; i++,j--) {
            // H(i, j + 1) = H(i, j) * R + H(j + 1, j + 1)
            k = k * 31 + s.charAt(i);
            // H(i - 1, j) = H(i, j) + H(i - 1, i - 1) * R ^ (j - i + 1)
            n = n + s.charAt(j) * m;

            m *= a;

            if (k == n) {
                length = i + 1;
            }

        }

        return s.substring(0, length);
    }

}
```

