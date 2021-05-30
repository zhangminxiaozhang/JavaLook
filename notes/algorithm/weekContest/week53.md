#### [5754. 长度为三且各字符不同的子字符串](https://leetcode-cn.com/problems/substrings-of-size-three-with-distinct-characters/)

<font size=4>**题意:**</font>

```
给定一个字符串，对于每个子串长度是3，如果没有重复字母的话就算有效，求出有多少个有效的？

输入：s = "xyzzaz"
输出：1
解释：总共有 4 个长度为 3 的子字符串："xyz"，"yzz"，"zza" 和 "zaz" 。
唯一的长度为 3 的好子字符串是 "xyz" 。

输入：s = "aababcabc"
输出：4
解释：总共有 7 个长度为 3 的子字符串："aab"，"aba"，"bab"，"abc"，"bca"，"cab" 和 "abc" 。
好子字符串包括 "abc"，"bca"，"cab" 和 "abc" 。
```

<font size=4>**题解：**</font>

```java
class Solution {
    public int countGoodSubstrings(String s) {
        int count=0;
        for(int i=0;i<s.length()-2;i++){
            String sub=s.substring(i,i+3);
            if(isOdd(sub)){
                count++;
            }
        }
        return count;
    }
    //判断是否有重复字符
    public boolean isOdd(String s){
        Set<Character> set=new HashSet();
        for(Character c:s.toCharArray()){
            if(set.contains(c)){
                return false;
            }
            set.add(c);
        }
        return true;
    }
}
```

#### [5755. 数组中最大数对和的最小值](https://leetcode-cn.com/problems/minimize-maximum-pair-sum-in-array/)

<font size=4>**题意:**</font>

```
给定长度是2N的数组，现划分成N小组，每小组2个数字，尽可能使每小组和最小，然后所有小组中的最大值，也就是最大化的最小和
输入：nums = [3,5,2,3]
输出：7
解释：数组中的元素可以分为数对 (3,3) 和 (5,2) 。
最大数对和为 max(3+3, 5+2) = max(6, 7) = 7 。

输入：nums = [3,5,4,2,4,6]
输出：8
解释：数组中的元素可以分为数对 (3,5)，(4,4) 和 (6,2) 。
最大数对和为 max(3+5, 4+4, 6+2) = max(8, 8, 8) = 8 。
```

<font size=4>**思路：**</font>

```
先对数组排序，然后将最大数+最小数字两两相加，这样加过后的最大值也就是整体的最大化的最小和
因为排过序的数组，如果最后一个数+第一个数不是最小和的话，拿其他的任意一个，可以发现都会比刚才的和要大，因此得出结论
A[i]+A[A.length-i-1]是最小和
```

<font size=4>**题解：**</font>

```java
class Solution {
    public int minPairSum(int[] nums) {
        Arrays.sort(nums);
        int min=0;
        for(int i=0;i<nums.length/2;i++){
            min=Math.max(min,nums[i]+nums[nums.length-i-1]);
        }
        return min;
    }
}
```

#### [5757. 矩阵中最大的三个菱形和](https://leetcode-cn.com/problems/get-biggest-three-rhombus-sums-in-a-grid/)

<font size=4>**题意:**</font>

```
给出一个二维矩阵，然后以任意点开始画出菱形，求出所有存在的菱形，将它们的边经过的点累加起来，求出最多不超过三个的累加和。

输入：grid = [[3,4,5,1,3],[3,3,4,2,3],[20,30,200,40,10],[1,5,5,4,1],[4,3,2,2,5]]
输出：[228,216,211]
解释：最大的三个菱形和如上图所示。
- 蓝色：20 + 3 + 200 + 5 = 228
- 红色：200 + 2 + 10 + 4 = 216
- 绿色：5 + 200 + 4 + 2 = 211
```

<div align="center">  <img src="https://i.bmp.ovh/imgs/2021/05/5b1923929b144b92.png" width="200"/> </div><br>

<font size=4>**思路：**</font>

```
首先对每个数字进行长度为 1，2，3 . . . 的左右斜线和进行储存

之后我们用 dp[i][j][cnt][0] 表示以 (i , j)为起点长度为cnt 的左斜线和。dp[i][j][cnt][1] 表示以 (i , j)为起点长度为cnt 的右斜线和

如果当前位置是  (i , j)，我们想求以 (i , j) 为起点长度为 cnt 的四角形和。sum = dp[i][j][cnt][0] + dp[i][j][cnt][1] + dp[i+cnt][j-cnt][cnt][1] + dp[i+cnt][j+cnt][cnt][0] - mat[i][j] - mat[i+2*cnt][j] - mat[i+cnt][j-cnt] - mat[i+cnt][j+cnt]。

//假设以下数组

//0 2 0 
//3 0 5     如果我们求长度是1的四角新，这里的例子由 （2 3 4 5）组成，它的和是 14 
//0 4 0     dp[0][1][1][0] = 2+3 = 5   dp[0][1][1][1] = 2+5 = 7
//          dp[1][0][1][1] = 3+4 = 7   dp[1][2][1][0] = 5+4 = 9
//          由于四个角被重复计算，我们需要最后把他们减去
//          (2+3) + (3+4) + (2+5) + (5 + 4) - (2 + 3 + 4 + 5) = 28 - 14 = 14

//简单来说，就是把四角形的四条线加起来
```

<font size=4>**题解：**</font>

```java
class Solution {
    public int[] getBiggestThree(int[][] grid) {
        int n=grid.length;
        int m=grid[0].length;
        //dp[i][j][count][0]表示以（i，j）为起点长度是count的左斜线的和
        int[][][][] dp=new int[n][m][Math.max(n,m)+1][2];
        Set<Integer> set=new TreeSet();
        for(int i=0;i<n;i++){
            for(int j=0;j<m;j++){
                 //求左斜线的和
                int r=i,c=j;
                int sum=0;
                int count=0;
                while(r<n&&c>=0){
                    sum+=grid[r++][c--];
                    dp[i][j][count++][0]=sum;
                }
                 //求右斜线的和
                 r=i;
                 c=j;
                 sum=0;
                 count=0;
                 while(r<n&&c<m){
                     sum+=grid[r++][c++];
                     dp[i][j][count++][1]=sum;
                 }
            }
        }
        //开始计算菱形的边和
        for(int i=0;i<n;i++){
            for(int j=0;j<m;j++){
                //存储长度为0的菱形
                set.add(grid[i][j]);
                for(int cnt=1;cnt<=n;cnt++){
                    //判断是否越界
                    if(i+cnt*2>=n||j-cnt<0|j+cnt>=m){
                        break;
                    }
                    int sum=dp[i][j][cnt][0]+dp[i][j][cnt][1]+dp[i+cnt][j-cnt][cnt][1]+dp[i+cnt][j+cnt][cnt][0]-grid[i][j]-grid[i+2*cnt][j]-grid[i+cnt][j+cnt]-grid[i+cnt][j-cnt];
                    set.add(sum);
                }
            }
        }
        //对set中最多前三个数进行取出
      List<Integer> list= new ArrayList(set);
      Collections.reverse(list);
      int[] res=new int[Math.min(3,list.size())];
      for(int i=0;i<res.length;i++){
          res[i]=list.get(i);
      }
      return res;
    }
}
```

第四题，放弃了。。

第三题解题参考来自https://mp.weixin.qq.com/s/4-XkvDVoGapYWo4Px40kvQ

