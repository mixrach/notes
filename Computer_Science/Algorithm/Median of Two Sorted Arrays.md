题目: There are two sorted arrays **nums1** and **nums2** of size m and n respectively.

Find the median of the two sorted arrays. The overall run time complexity should be O(log (m+n)).

You may assume **nums1** and **nums2** cannot be both empty.

对于跟定的有序数组A和B，寻找两个数组的中位数。A和B不会同时为空。



我们假设A 为大小为N的数组，B为大小为M的数组，且N<=M; 则其中的一半为H = (N+M)/2；

以A数组为例，如果N为奇数，则中位数为A[N/2], 如果N为偶数，则中位数为A[N/2] + A[N/2 -1]



对于两个数组组合起来的数组，如果N+M为偶数，则我们关心四个值，反之我们关心两个值。下面具体说关心的数值。



我们假设A的前I个元素处于左半区，则B中必然有H - I个元素处于左半区，这里不论N+M为偶数还是奇数。

我们令J = H - I

不同的是，我们中位数怎么得来。这里我们假设I不会超过N，即一定存在一个小于N的数值i，使得A[0..i-1]处于合并数组的左半区。

当N+M为偶数时，处于左边的数值为 A[0...i-1], B[0..j -1];  处于右边的值为 A[i..N-1] B[j..M-1]，中位数为这左边中最大值与右边最小值的平均值。

当N+M为奇数时，我们知道A[0..i-1], B[0..j-1]全部属于左半区，而中位数是A[i] 和A[j]中较小那一个。

以上我们假设的是存在数值I能够将A拆分为两个数组，显然中位数大于A中所有的数，也有可能小于A中所有的数。

当A[0..N-1]全部属于左半区，即I = n， 则右半区最小值为B[j] 

当A[0...N-1]全部属于右半区，即I= 0， 则左半区最大值为B[j -1]  

上述之所以只能确定一侧的值，主要是因为M=N的情况。

如果M = N， 第一种情况j为0， 则不能确定左半区的最大值为B[j -1]，第二种情况j = M，不能确定右半区最小值为B[j]。



由上述分析，我们可以知道，关键的问题是寻找数字I，使得要么I等于0,要么等于N，要么满足

A[i -1] < B[j],且 A[i]>B[j-1];

寻找这样一个值最容易想到的必然是二分查找，这样可以把时间控制在logN，也满足题目的要求。



我们分析一下二分查找的条件。

边界条件下，如果I == 0或 I==N 则终止，这个条件可以分别写成 i 满足

i>0 与 i <n

当我们发现A[i-1] > B[j]时，说明当前的i太大了我们将0->i-1；

当我们发现A[i] < B[j-1]时，说明当前的i太小了，我们应该寻找i +1 -> N

经过以上分析，我们有如下代码

```C++
class Solution
{
    public:
        double findMedianSortedArrays(vector<int> &nums1, vector<int> &nums2)
        {

            int m = nums1.size();
            int n = nums2.size();

            if(m == 0) return getMid(nums2);
            if(n == 0) return getMid(nums1);

            if(m > n)
            {
                nums1.swap(nums2);
                m = m^n;
                n = m^n;
                m = m^n;
            }

            int minIndex = 0, maxIndex = m;
            int half = (m + n) / 2;
            while(minIndex <= maxIndex)
            {
                int i = (minIndex + maxIndex) / 2;
                int j = half - i;
                if(i < maxIndex && nums1[i] < nums2[j - 1])
                {
                    minIndex = i + 1;
                }
                else if(i > minIndex && nums1[i - 1] > nums2[j])
                {
                    maxIndex = i - 1;
                }
                else
                {
                    int minRight = 0;
                    if(i == m) minRight = nums2[j];
                    else if(j == n) minRight = nums1[i];
                    else minRight = getMin(nums2[j], nums1[i]);

                    if((m + n) % 2 == 1) return minRight;
                    
                    int maxLeft = 0;
                    if(i == 0) maxLeft = nums2[j - 1];
                    else if(j == 0) maxLeft = nums1[i - 1];
                    else maxLeft = getMax(nums2[j - 1], nums1[i - 1]);

                    return (minRight + maxLeft) / 2.0;
                }
            }
            return 0.0;
        }

    public:
        int getMax(int i, int j)
        {
            if(i < j) return j;
            else return i;
        }

    public:
        int getMin(int i, int j)
        {
            if(i < j) return i;
            else return j;
        }


    public :
        double getMid(vector<int> &nums)
        {
            int m = nums.size();
            if(m % 2 == 0)
            {
                return (nums[m / 2] + nums[m / 2 - 1]) / 2.0;
            }
            else
            {
                return nums[m / 2];
            }
        }
};
```





