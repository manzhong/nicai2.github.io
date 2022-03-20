---
title: LeetCode
tags: 数据结构与算法
categories: 数据结构与算法
abbrlink: 61255
date: 2020-02-26 19:35:41
summary_img:
encrypt:
enc_pwd:
---

### 简介

​	个人刷题思路记录，每日一题！！！！！时刻保持竞争力与思维活跃

### 一 数组

##### 1.1 给定一个只包含正整数的非空数组。是否可以将这个数组分割成两个子集，使得两个子集的元素和相等。

```java
public class ArraySplitArrsumEq {
//给定一个只包含正整数的非空数组。是否可以将这个数组分割成两个子集，使得两个子集的元素和相等。
    public static void main(String[] args) {
        int[] arr = new int[]{1,2,3,4,5,6,7};
        boolean b = canPartition(arr);
        System.out.println(b);
    }
    public static boolean canPartition(int[] nums) {
        boolean flag =false;
        if(nums.length<=1){
            return flag;
        }
        int sum=0;
        for(int i=0;i<nums.length;i++){
            sum = sum+nums[i];
        }
        //判断奇偶性 两个子集的元素和相等 则元素和定为偶数
        if(sum%2!=0){
            return flag;
        }
        //排序
        Arrays.sort(nums);
        int i =0,j=nums.length-1;
        int p =sum/2;
        int sum1 =0;
        //判段最大值是否大于元素和的一半
        if(nums[j]>p){
            return false;
        }
        //刚好的等于则一定可以
        if(nums[j]==p){
            return true;
        }
        //其他情况判断 最大值小于元素和的一半
      	// 每次循环查找 小于最大值和元素和的差值 
        //最大值和元素和的差值
        p=p-nums[j]; //7
        for(int k=nums.length-2;k>=0;k--){
            if(nums[k]<=p){
                int copy=p;
                copy = copy -nums[k];
                if(copy==0){
                    return true;
                }
                for(int l=k-1;l>=0;l--){
                    if(nums[l]<=copy){
                        copy=copy-nums[l];
                        if(copy==0){
                            return true;
                        }
                    }
                }
            }
        }
        return false;
    }
}
```

##### 1.2 位运算符系列

**位运算符简介**

- 在Java语言中，提供了7种位运算符，分别是按位与（&）、按位或（|）、按位异或（^）、取反(~)、左移(<<)、带符号右移(>>)和无符号右移(>>>)。这些运算符当中，仅有~是单目运算符，其他运算符均为双目运算符.
- 一个常识，那就是：位运算符是对long、int、short、byte和char这5种类型的数据进行运算的，我们不能对double、float和boolean进行位运算操作

**使用简介**

```java
//按位与 & 转为二进制 按位与 如果两个二进制位上的数都是1，那么运算结果为1，其他情况运算结果均为0
        System.out.println(2&2); //2
        System.out.println(5&2); //0
        System.out.println(0&2); //0
        System.out.println(0&0); //0
        //按位或 | 转为二进制 按位或 如果两个二进制数都是0，计算结果为0，其他情况计算结果均为1
        System.out.println(2|2|2); //2
        System.out.println(1|2); //3
        System.out.println(5|2); //7
        System.out.println(10|3); //11
        System.out.println(2|5); //7
        System.out.println(0|2); //2
        System.out.println(0|0); //0
        //异或 两个二进制位上的数字如果相同，则运算结果为0，如果两个二进制位上的数字不相同，则运算结果为1 a^b与b^a是等价的，虽然a和b交换了位置，但还是会运算出相同的结果
        //a^b^a等于b
        System.out.println(2^2); //0
        System.out.println(5^2); //7
        System.out.println(0^2); //2
        System.out.println(0^0); //0
        //按位取反 首先把数字5转换成补码形式，之后把每个二进制位上的数字进行取反，如果是0就变成1，如果1就变成0，经过取反后得到的二进制串就是运算结果
        System.out.println(~2); //-3
        System.out.println(~3); //-4
        System.out.println(~4); //-5
        System.out.println(~0); //-1
        // 左移 <<   a<<b 等价于 a* (2的b次方)
        System.out.println(3<<2); //12 等价于 3* 2的平方
        System.out.println(3<<4); //48 等价于 3* 2的4次方
        // 带符号右移 >>
        System.out.println(3>>2); //0 等价于 3/ 2的平方 取整
        System.out.println(15>>3); //1 等价于 15/ 2的3次方 取整
        System.out.println(24>>3); //3 等价于 24/ 2的3次方 取整
        System.out.println(-24>>3); //-3 等价于 -24/ 2的3次方 取整
        // 无符号右移 >>> 正数与>> 同 负数>>>会得到一个正数
        System.out.println(3>>>2); //0 等价于 3/ 2的平方 取整
        System.out.println(15>>>3); //1 等价于 15/ 2的3次方 取整
        System.out.println(24>>>3); //3 等价于 24/ 2的3次方 取整
        System.out.println(-24>>>3); //536870909
```

```java
// 1.给定一个非空整数数组，除了某个元素只出现一次以外，其余每个元素均出现两次。找出那个只出现了一次的元素。
public static void main(String[] args) {
        int[] num = {1,2,1,3,2,3,4,4,4} ;
        int only = 0;
        for(int i =0 ; i< num.length;i++){
            only = only^num[i];
        }
        System.out.println(only);
 }
```

##### 1.3 给定一个大小为 n 的数组，找到其中的多数元素。多数元素是指在数组中出现次数 大于 ⌊ n/2 ⌋ 的元素。你可以假设数组是非空的，并且给定的数组总是存在多数元素。

```java
// 摩尔投票法  (又称同归于尽法) 摩尔投票法的一大应用就是求众数。
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class Moer {

    // 给定一个大小为 n 的数组，找到其中的多数元素。多数元素是指在数组中出现次数 大于 ⌊ n/2 ⌋ 的元素 该元素只能有一个
    // 就是对杀 不同的数字对消 最后统计count>0的数字
    public static int get2n(int[] nums){
        int maj=nums[0],count=1;
        for (int j=1 ; j<nums.length;j++){
            if(maj==nums[j]){
                count++;
            }else
                count--;
            if(count==0){
                maj=nums[j];
                count=1;
            }
        }
        return maj;
    }

    // 给定一个大小为 n 的数组，找到其中的多数元素。多数元素是指在数组中出现次数 大于 ⌊ n/3 ⌋ 的元素 该元素最多2个
    // 此时只需要设置maj1和count1、maj2和count2来分别记录这两个元素的抵消情况即可。如果出现了和maj1或maj2相同的元素，那么对应的count1和count2就自加1，如果元素与maj1和maj2都不相同，那么count1和count2就都应当自减1，如果maj1或maj2抵消掉后，就应当更新对应的maj1或maj2
    public static ArrayList<Integer> get3n(int[] array){
        ArrayList<Integer> list = new ArrayList<Integer>();
        if(array == null ||array.length == 0) return null;
        int maj1 = array[0],maj2 = array[0];
        int count1 = 0,count2 = 0;
        int len = array.length;
        // 判断刚好有两个值满足是
        for(int i = 0;i<len;i++) {
            if(array[i] == maj1) count1++;
            else if(array[i] == maj2) count2++;
            else if(count1<=0) {
                count1 = 1;
                maj1 = array[i];
            }else if(count2<=0) {
                count2 = 1;
                maj2 = array[i];
            }else {
                --count1;
                --count2;
            }
        }
        // 再次判断 是否符合 因为当只有一个值超过n/3时会出问题
        count1 = count2 =0;
        for(int i =0;i<len;i++) {
            if(array[i] == maj1) count1++;
            if(array[i] == maj2) count2++;
        }
        if(count1>len/3) list.add(maj1);
        // 第二个值还要判断 两个值是否相等 相等只需传一个 相等是(数组都是同一个值)
        if(count2>len/3 & maj2!=maj1) list.add(maj2);
        return list;
    }

    // n/k 问题
    public static void solve3(int [] arr,int k) {
        Map<Integer,Integer> map = new HashMap<Integer,Integer>();
        //首先找到k-1个不同的数，并记录他们出现的次数，继续向后遍历
        //若下一个数map里有则将该数个数即value+1,否则map里数的个数都减1
        //否则加入该数，次数记为1
        for(int anArr:arr) {
            Integer value;
            if((value = map.get(anArr))!=null) {
                map.replace(anArr, value+1);
            }else {
                if(map.size() == k-1) {
                    subtrackAll(map,k);
                }else {
                    map.put(anArr,1);
                }
            }
        }
        //将map里所有数的个数即value都清0，因为map中的数并非都符合要求
        map.keySet().forEach(integer->map.replace(integer, 0));
        //再次循环统计map里数字的真正个数
        for(int num:arr) {
            Integer val;
            if((val = map.get(num))!= null) {
                map.replace(num, val+1);
            }
        }
        map.forEach((key,value)->{
            if(value>arr.length/k) {
                System.out.println(key+" ");
            }
        });
    }
    private static void subtrackAll(Map<Integer, Integer> map, int k) {
        List<Integer> list = new ArrayList();
        // 将map里全部数字的value值减1，减1后若value==0，则删除该键
        map.forEach((integer,integer2)->{
            if(integer2 == 1) {
                list.add(integer);
            }else map.replace(integer, integer2-1);
        });
        if(list.size()!=0) {
            for(Integer key:list) {
                map.remove(key);
            }
        }
    }
    public static void main(String[] args) {
        int[] num2 ={1,1,-3,2,1};
        System.out.println(get2n(num2));

        int[] num4 = {1,1,1,1,1};
        int[] num3 ={4,1,1,1,1,2,1,2,2,2,2,3,3,4};
        System.out.println(get3n(num3));

        solve3(num2,3);
    }
}
```

##### 1.4 编写一个高效的算法来搜索 m x n 矩阵 matrix 中的一个目标值 target 。该矩阵具有以下特性：每行的元素从左到右升序排列。每列的元素从上到下升序排列。

```
1 2  3  4  5
6 10 14 18 22
7 11 15 19 23
8 12 16 20 24
9 13 17 21 25
// 从右上角来看就是个二叉搜索树 小的在左 大的在右
```

```java
// 从右上开始
public boolean searchMatrix(int[][] matrix, int target) {
        for (int i = 0; i < matrix.length; i++) {
            for (int j = matrix[0].length - 1; j >= 0; j--) {
                if(target < matrix[i][j])continue;
                else if(target  == matrix[i][j]) return true;
                else break;
            }
        }
        return false;
   }
```







### 二 链表

##### 2.1 给你一个链表，删除链表的倒数第 `n` 个结点，并且返回链表的头结点。

```java
class Solution {
    public ListNode removeNthFromEnd(ListNode head, int n) {
        // 双指针的思想
        // 定义一个快指针,定义一个慢指针
        ListNode slow = head;
        ListNode fast = head;
        // 先让快指针走n步
        while(n--!=0){
            fast=fast.next;
        }
        // 如果快指针走到了最后说明删除的是第一个节点,就返回head.next就好
        if(fast==null){
            return head.next;
        }
        // 使得slow每次都是在待删除的前一个节点, 所以要先让fast先走一步
        fast=fast.next;
        while(fast!=null){
            fast=fast.next;
            slow=slow.next;
        }
        // 因为已经保证了是待删除节点的前一个节点, 直接删除即可
        slow.next=slow.next.next;
        return head;
    }
}
```

##### 2.2 求单链表中节点的个数

```java
//注意检查链表是否为空。时间复杂度为O（n）
//方法：获取单链表的长度
    public int getLength(Node head) {
        if (head == null) {
            return 0;
        }

        int length = 0;
        Node current = head;
        while (current != null) {
            length++;
            current = current.next;
        }

        return length;
    }
```

##### 2.3 查找单链表中的倒数第k个结点

```java
// 方法 1 O(n)
public int findLastNode(int index) {  //index代表的是倒数第index的那个结点

        //第一次遍历，得到链表的长度size
        if (head == null) {
            return -1;
        }

        current = head;
        while (current != null) {
            size++;
            current = current.next;
        }

        //第二次遍历，输出倒数第index个结点的数据
        current = head;
        for (int i = 0; i < size - index; i++) {
            current = current.next;
        }

        return current.data;
    }
// 方法2 与第一题类似 快慢指针 该种不需遍历链表的长度
public Node findLastNode(Node head, int k) {
        if (k == 0 || head == null) {
            return null;
        }

        Node first = head;
        Node second = head;

        //让second结点往后挪k-1个位置
        for (int i = 0; i < k - 1; i++) {
            System.out.println("i的值是" + i);
            second = second.next;
            if (second == null) { //说明k的值已经大于链表的长度了
                //throw new NullPointerException("链表的长度小于" + k); //我们自己抛出异常，给用户以提示
                return null;
            }
        }

        //让first和second结点整体向后移动，直到second走到最后一个结点
        while (second.next != null) {
            first = first.next;
            second = second.next;
        }

        //当second结点走到最后一个节点的时候，此时first指向的结点就是我们要找的结点
        return first;
    }
```

##### 2.4 查找单链表中的中间节点

**不允许算链表的长度**

```java
// 思路 : 也是设置两个指针first和second，只不过这里是，两个指针同时向前走，second指针每次走两步，first指针每次走一步，直到second指针走到最后一个结点时，此时first指针所指的结点就是中间结点。注意链表为空，链表结点个数为1和2的情况。时间复杂度为O（n）
//方法：查找链表的中间结点
    public Node findMidNode(Node head) {

        if (head == null) {
            return null;
        }

        Node first = head;
        Node second = head;
        //每次移动时，让second结点移动两位，first结点移动一位
        while (second != null && second.next != null) {
            first = first.next;
            second = second.next.next;
        }

        //直到second结点移动到null时，此时first指针指向的位置就是中间结点的位置
        return first;
    }
```

##### 2.5 合并两个有序的单链表，合并之后的链表依然有序

```java
public class MergeListTest {

    public static class Node{

        int data;
        Node next;
        public Node(int data) {
            super();
            this.data = data;
        }
    }
    /**
     * 递归方式合并两个单链表
     * @param head1 有序链表1
     * @param head2 有序链表2
     * @return 合并后的链表
     */

    public static Node mergeTwoList(Node head1, Node head2) {
        //递归结束条件
        if (head1 == null && head2 == null) {
            return null;
        }
        if (head1 == null) {
            return head2;
        }
        if (head2 == null) {
            return head1;
        }
        //合并后的链表
        Node head = null;
        if (head1.data > head2.data) {
            //把head较小的结点给头结点
            head = head2;
            //继续递归head2
            head.next = mergeTwoList(head1, head2.next);
        } else {
            head = head1;
            head.next = mergeTwoList(head1.next, head2);

        }
        return head;
    }

    /**
     * 非递归方式
     * @param head1 有序单链表1
     * @param head2 有序单链表2
     * @return 合并后的单链表
     */

    public static Node mergeTwoList2(Node head1, Node head2) {
        if (head1 == null || head2 == null) {
            return head1 != null ? head1 : head2;
        }
        //合并后单链表头结点
        Node head = head1.data < head2.data ? head1 : head2;
        Node cur1 = head == head1 ? head1 : head2;
        Node cur2 = head == head1 ? head2 : head1;
        Node pre = null;//cur1前一个元素
        Node next = null;//cur2的后一个元素
        while (cur1 != null && cur2 != null) {
            //第一次进来肯定走这里
            if (cur1.data <= cur2.data) {
                pre = cur1;
                cur1 = cur1.next;
            } else {
                next = cur2.next;
                pre.next = cur2;
                cur2.next = cur1;
                pre = cur2;
                cur2 = next;
            }

        }
        pre.next = cur1 == null ? cur2 : cur1;
        return head;
    }

    public static void main(String[] args) {
        Node node1 = new Node(1);
        Node node2 = new Node(2);
        Node node3 = new Node(3);
        Node node4 = new Node(4);
        Node node5 = new Node(5);
        node1.next = node3;
        node3.next = node5;
        node2.next = node4;
        Node node = mergeTwoList(node1, node2);
        //Node node = mergeTwoList2(node2, node1);
        while (node != null) {
            System.out.print(node.data + " ");
            node = node.next;
        }

    }
}
```

##### 2.6 单链表的反转*

递归法: 重点理解

```java
// 方法1 
//方法：链表的反转
    public Node reverseList(Node head) {
        //如果链表为空或者只有一个节点，无需反转，直接返回原链表的头结点
        if (head == null || head.next == null) {
            return head;
        }
        Node current = head;
        Node next = null; //定义当前结点的下一个结点
        Node reverseHead = null;  //反转后新链表的表头
        while (current != null) {
            next = current.next;  //暂时保存住当前结点的下一个结点，因为下一次要用
            current.next = reverseHead; //将current的下一个结点指向新链表的头结点
            reverseHead = current;
            current = next;   // 操作结束后，current节点后移
        }
        return reverseHead;
    }

// 方法2 递归 
    public static Node reverseList2(Node head){
        //如果链表为空或者只有一个节点，无需反转，直接返回原链表的头结点
        if (head == null || head.next == null) {
            return head;
        }
        Node last=reverseList2(head.next);
        head.next.next = head;
        head.next = null;
        return last;
    }
// 测试
public static void main(String[] args) {
        Node node1 = new Node(1);
        Node node2 = new Node(2);
        Node node3 = new Node(3);
        Node node4 = new Node(4);
        Node node5 = new Node(5);
        node1.next = node3;
        node3.next = node5;
        Node nodere=reverseList(node1);
        while (nodere != null) {
            System.out.print(nodere.data + " ");
            nodere = nodere.next;
        }
    }
```

**递归法详解:**

```
如这样一个单链表 [1] -> [2] -> [3] ->[4]
首先函数执行,if语句不满足,向下执行到递归语句
Node last=reverseList2(head.next);
则一直递归直到递归结束(期间一直执行的语句就是if判断语句),则最后一次传入的是[4],if判断语句,判断不符合,返回[4]的节点,然后递归开始回执,回执1次,执行[3]->[4] 的情形 最终 null<-[3]<-[4],再次回执,传入 [2]-> [3](-> null) <-[4],然后执行后 null<-[2]<-[3]<-[4],.....直到最后回执结束,返回[4]的节点(last),最终程序执行完毕,生成
null<-[1] <- [2] <- [3] <- [4]
```

##### 2.7 从尾到头打印单链表

```java
// 方法1 
//方法：从尾到头打印单链表
    public void reversePrint(Node head) {
        if (head == null) {
            return;
        }
        Stack<Node> stack = new Stack<Node>();  //新建一个栈
        Node current = head;
        //将链表的所有结点压栈
        while (current != null) {-
            stack.push(current);  //将当前结点压栈
            current = current.next;
        }
        //将栈中的结点打印输出即可
        while (stack.size() > 0) {
            System.out.println(stack.pop().data);  //出栈操作
        }
    }
// 方法2 递归 当链表很长的时候，就会导致方法调用的层级很深，有可能造成栈溢出。而方法1的显式用栈，是基于循环实现的，代码的鲁棒性要更好一些。
public void reversePrint(Node head) {
        if (head == null) {
            return;
        }
        reversePrint(head.next);
        System.out.println(head.data);
}
```

##### 2.8 判断单链表是否有环*

​	这里也是用到两个指针，如果一个链表有环，那么用一个指针去遍历，是永远走不到头的。因此，我们用两个指针去遍历：first指针每次走一步，second指针每次走两步，如果first指针和second指针相遇，说明有环。时间复杂度为O (n)。

```java
// 方法1 快慢指针 一个一次一步,一个一次两步,循环遍历,当两者相等,有环
public class MergeListTest {
      public static class Node{
        int data;
        Node next;
        public Node(int data) {
            super();
            this.data = data;
        }
    }
    //方法：判断单链表是否有环
    public static boolean hasCycle(Node head) {

        if (head == null) {
            return false;
        }
        Node first = head;
        Node second = head;

        while (second != null) {
            first = first.next;   //first指针走一步
            second = second.next.next; // second指针走两步
            if (first == second) {  //一旦两个指针相遇，说明链表是有环的
                return true;
            }
        }
        return false;
    }
   public static void main(String[] args) {
        Node node1 = new Node(1);
        Node node2 = new Node(2);
        Node node3 = new Node(3);
        Node node4 = new Node(4);
        Node node5 = new Node(5);
        node1.next = node3;
        node3.next = node5;
        node2.next = node4;
        //判断链表是否有环
        // node1加环
        node5.next = node3;
        System.out.println(hasCycle(node2));

    }
}
```

##### 2.9 取出有环链表中，环的长度

```
// 环的类型
[1] -> [2] -> [3] ->[4] -> [1] 1234 一个环
// 或
[1] -> [2] -> [3] ->[4] -> [2]  234 一个环
```

```java
// 利用上边 hasCycle方法,这个方法的返回值是boolean型，但是现在要把这个方法稍做修改，让其返回值为相遇的那个结点。然后，我们拿到这个相遇的结点就好办了，这个结点肯定是在环里嘛，我们可以让这个结点对应的指针一直往下走，直到它回到原点，就可以算出环的长度了。
public class MergeListTest {
    public static class Node{
        int data;
        Node next;
        public Node(int data) {
            super();
            this.data = data;
        }
    }

    //方法：判断单链表是否有环。返回的结点是相遇的那个结点 若无环 则返回null
    public static Node hasCycleReturnNode(Node head) {

        if (head == null) {
            return null;
        }
        Node first = head;
        Node second = head;
        while (second != null) {
            first = first.next;
            second = second.next.next;
            if (first == second) {  //一旦两个指针相遇，说明链表是有环的
                return first;  //将相遇的那个结点进行返回
            }
        }
        return null;
    }

    //方法：有环链表中，获取环的长度。参数node代表的是相遇的那个结点
    public static int getCycleLength(Node node) {
        if (node == null) {
            return 0;
        }
        Node current = node;
        int length = 0;

        while (current != null) {
            current = current.next;
            length++;
            if (current == node) {  //当current结点走到原点的时候
                return length;
            }
        }
        return length;
    }

    public static void main(String[] args) {
        Node node1 = new Node(1);
        Node node2 = new Node(2);
        Node node3 = new Node(3);
        Node node4 = new Node(4);
        Node node5 = new Node(5);
        node1.next = node3;
        node3.next = node5;
        node2.next = node4;
        // node1加环
        node5.next = node3;
      	// 打印环长度
        System.out.println(getCycleLength(hasCycleReturnNode(node1)));
    }
}

```

##### 2.10 单链表中，取出环的起始点

```
// 环的类型
[1] -> [2] -> [3] ->[4] -> [1] 环起始点[1]
// 或
[1] -> [2] -> [3] ->[4] -> [2]  环起始点 [2]
```

**实现:**

```java
//取出环的长度的方法getCycleLength，用这个方法来获取环的长度length。拿到环的长度length之后，需要用到两个指针变量first和second，先让second指针走length步；然后让first指针和second指针同时各走一步，当两个指针相遇时，相遇时的结点就是环的起始点。
//注：为了找到环的起始点，我们需要先获取环的长度，而为了获取环的长度，我们需要先判断是否有环。所以这里面其实是用到了三个方法。
public class MergeListTest {
    public static class Node{

        int data;
        Node next;
        public Node(int data) {
            super();
            this.data = data;
        }
    }
    //方法：判断单链表是否有环
    public static boolean hasCycle(Node head) {

        if (head == null) {
            return false;
        }
        Node first = head;
        Node second = head;

        while (second != null) {
            first = first.next;   //first指针走一步
            second = second.next.next; // second指针走两步
            if (first == second) {  //一旦两个指针相遇，说明链表是有环的
                return true;
            }
        }
        return false;
    }
    //方法：判断单链表是否有环。返回的结点是相遇的那个结点 若无环 则返回null
    public static Node hasCycleReturnNode(Node head) {

        if (head == null) {
            return null;
        }

        Node first = head;
        Node second = head;

        while (second != null) {
            first = first.next;
            second = second.next.next;
            if (first == second) {  //一旦两个指针相遇，说明链表是有环的
                return first;  //将相遇的那个结点进行返回
            }
        }
        return null;
    }

    //方法：有环链表中，获取环的长度。参数node代表的是相遇的那个结点
    public static int getCycleLength(Node node) {

        if (node == null) {
            return 0;
        }

        Node current = node;
        int length = 0;

        while (current != null) {
            current = current.next;
            length++;
            if (current == node) {  //当current结点走到原点的时候
                return length;
            }
        }

        return length;
    }
    //方法：获取环的起始点。参数length表示环的长度
    public static Node getCycleStart(Node head, int cycleLength) {
        if (head == null) {
            return null;
        }
        Node first = head;
        Node second = head;
        //先让second指针走length步
        for (int i = 0; i < cycleLength; i++) {
            second = second.next;
        }
        //然后让first指针和second指针同时各走一步
        while (first != null && second != null) {
            first = first.next;
            second = second.next;

            if (first == second) { //如果两个指针相遇了，说明这个结点就是环的起始点
                return first;
            }
        }
        return null;
    }

    public static void main(String[] args) {
        Node node1 = new Node(1);
        Node node2 = new Node(2);
        Node node3 = new Node(3);
        Node node4 = new Node(4);
        Node node5 = new Node(5);
        node1.next = node3;
        node3.next = node5;
        node2.next = node4;

        //判断链表是否有环
        // node1加环
        node5.next = node3;
        //判断是否有环
        System.out.println(hasCycle(node1));
        // 获取环长度
        System.out.println(getCycleLength(hasCycleReturnNode(node1)));
        // 获取环的起始点
     	System.out.println(getCycleStart(node1,getCycleLength(hasCycleReturnNode(node1))).data);
    }
}
```

##### 2.11 判断两个单链表相交的第一个交点

```java
public class MergeListTest {

    public static class Node{

        int data;
        Node next;
        public Node(int data) {
            super();
            this.data = data;
        }
    }

    //方法：获取单链表的长度
    public static int getLength(Node head) {
        if (head == null) {
            return 0;
        }
        int length = 0;
        Node current = head;
        while (current != null) {
            length++;
            current = current.next;
        }
        return length;
    }
    //判断两个链表相交的第一个节点 堆栈方法 利用先进后出 
    public static Node twoListFirstNode1(Node head1,Node head2){
        if(head1 == null || head2 == null){
            return null;
        }
        Stack<Node> stack1 = new Stack<Node>();
        Stack<Node> stack2 = new Stack<Node>();
        Node h1 = head1;
        Node h2 = head2;
        while(h1 != null){
            stack1.push(h1);
            h1 = h1.next;
        }
        while(h2 != null){
            stack2.push(h2);
            h2 = h2.next;
        }
        Node first = null ;
        Node n1=stack1.pop();
        Node n2=stack2.pop();
        while(n1 == n2){
            first = n1;
            n1=stack1.pop();
            n2=stack2.pop();
        }
        return first;
    }
    //方法：求两个单链表相交的第一个交点 快慢指针
  //首先遍历两个链表得到它们的长度。在第二次遍历的时候，在较长的链表上走 |len1-len2| 步，接着再同时在两个链表上遍历，找到的第一个相同的结点就是它们的第一个交点。
    public static Node twoListFirstNode2(Node head1, Node head2) {
        if (head1 == null || head2 == null) {
            return null;
        }

        int length1 = getLength(head1);
        int length2 = getLength(head2);
        int lengthDif = 0;  //两个链表长度的差值

        Node longHead;
        Node shortHead;
        //找出较长的那个链表
        if (length1 > length2) {
            longHead = head1;
            shortHead = head2;
            lengthDif = length1 - length2;
        } else {
            longHead = head2;
            shortHead = head1;
            lengthDif = length2 - length1;
        }
        //将较长的那个链表的指针向前走length个距离
        for (int i = 0; i < lengthDif; i++) {
            longHead = longHead.next;
        }
        //将两个链表的指针同时向前移动
        while (longHead != null && shortHead != null) {
            if (longHead == shortHead) { //第一个相同的结点就是相交的第一个结点
                return longHead;
            }
            longHead = longHead.next;
            shortHead = shortHead.next;
        }
        return null;
    }

    public static void main(String[] args) {
        Node node1 = new Node(1);
        Node node2 = new Node(2);
        Node node3 = new Node(3);
        Node node4 = new Node(4);
        Node node5 = new Node(5);
        node1.next = node3;
        node3.next = node5;
        node2.next = node4;
        //判断链表的连接点
        // 构造连接点
        node4.next = node5;
        System.out.println(twoListFirstNode1(node1,node2).data);
        System.out.println(twoListFirstNode2(node1,node2).data);
    }
}
```

### 三 树



### 四 动态规划

##### 4.1 鸡蛋掉落问题

给你 k 枚相同的鸡蛋，并可以使用一栋从第 1 层到第 n 层共有 n 层楼的建筑。已知存在楼层 f ，满足 0 <= f <= n ，任何从 高于 f 的楼层落下的鸡蛋都会碎，从 f 楼层或比它低的楼层落下的鸡蛋都不会破。每次操作，你可以取一枚没有碎的鸡蛋并把它从任一楼层 x 扔下（满足 1 <= x <= n）。如果鸡蛋碎了，你就不能再次使用它。如果某枚鸡蛋扔下后没有摔碎，则可以在之后的操作中 重复使用 这枚鸡蛋。请你计算并返回要确定 f 确切的值 的 最小操作次数 是多少？

如:

```text
输入：k = 1, n = 2
输出：2
解释：
鸡蛋从 1 楼掉落。如果它碎了，肯定能得出 f = 0 。 
否则，鸡蛋从 2 楼掉落。如果它碎了，肯定能得出 f = 1 。 
如果它没碎，那么肯定能得出 f = 2 。 
因此，在最坏的情况下我们需要移动 2 次以确定 f 是多少。 
```





