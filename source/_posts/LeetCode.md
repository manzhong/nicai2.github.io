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

​	个人刷题思路记录

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



### 三 树



### 四 动态规划





