# [LeetCode]	链表




#### 声明：下面的所有代码都基于此链表类

```java
public class ListNode {
	int data;
	ListNode next;
}
```



## 1.删除某个节点

Question：给定一个链表，删除链表的倒数第n个节点，并返回链表的头节点。

思路：可以采用快慢指针，链表的很多问题都可以用快慢指针来解决。让快指针fast先走n-1步后，慢指针slow再开始走，当fast指向尾节点时，此时slow指向的就是要删除的节点。为了方便，我们可以创建一个哑节点。实际让fast先走n步，这样当fast指向尾节点时，slow指向的就是被删除节点的前一个节点，直接修改next指针就可以了。

```java
public ListNode removeFromLast(ListNode head, int n) {
        ListNode dummy = new ListNode(0);
        dummy.next = head;
        ListNode fast = dummy;
        ListNode slow = dummy;

        while (fast != null) {
            fast = fast.next;
            if (n-- < 0) {
                slow = slow.next;
            }
        }
        slow.next = slow.next.next;
        return dummy.next;
    }
```



## 2.判断链表中是否有环

Question：给定一个链表，判断链表中是否存在环。

思路：还记得我们初中数学学的环形操作追及问题吗？两个不同速度的人在一个环形的操场上奔跑终会相遇。因此此题可以用快慢指针来解决。快指针fast步长为2，慢指针slow步长为1，直到fast.next.next = null，如果fast和slow相遇，则说明链表中存在环。

```java
public boolean isCycle(Node head) {
        ListNode fast = head;
        ListNode slow = head;
        while (fast != null && fast.next != null) {
         	slow = slow.next;
            fast = fast.next.next;
            if (fast == slow) {
                return true;
            }
        }
        return false;
    }
```



## 3.求出环的长度

Question：拓展问题2，如果链表中存在环，请求出环的长度。

思路：fast和slow的每次相遇，fast就比slow多走了整整一圈。因此我们在slow和fast首次相遇时，slow呆在原地不动，让fast继续前进，当fast回到slow位置时，就是走了一圈，也就是环的长度。

```java
public int getLength(ListNode head){
        ListNode fast = head;
        ListNode slow = head;
        boolean isCycle = false;
        int len = 0;
        while (fast != null && fast.next != null){
            slow = slow.next;
            fast = fast.next.next;
            if (fast == slow){
                isCycle = true;
                break;
            }
        }

        if (isCycle){
            fast = fast.next;
            len++;
            while (fast != slow){
                fast = fast.next;
                len++;
            }
            return len;
        }
        return len;
    }
```



## 4.求入环点的位置

Question：拓展问题2，如果存在环，请求出入环点。

![未命名文件.png](https://i.loli.net/2020/01/16/Lnxky2tAlM3BSXZ.png)

思考：假设从链表头节点到入环点的距离为L，入环点到首次相遇点的距离为S₁，从首次相遇点回到入环点的距离为S₂。可以确定的是，当指针fast和slow首次相遇时，fast比slow多走了n(n ≧ 1)圈。由上图可知：当fast和slow首次相遇时，slow行进的距离是：L + S₁。fast行进的距离是：L + n(S₁ + S₂) + S₁。

假设fast步长为2，slow的步长为1。因为fast的速度是slow的2倍，那么同样的时间，fast所走的距离也是slow的2倍。由此可得：

&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;`` 2(L + S₁) = L + n(S₁ + S₂) + S₁`` 

整理式子得到：&emsp;&emsp;&emsp;``L = (n-1)(S₁ + S₂) + S₂``

也就是说从链表头节点到入环点的距离L，等于指针从首次相遇点绕环n-1圈，再回到入环点的距离。这样一来，在slow和fast首次相遇时，让其中一个指针回到头节点的位置，然后两个指针以步长为1的速度前进，最终相遇的地方就是入环点。

```java
public ListNode getCrosspoint(ListNode head) {
    ListNode fast = head;
    ListNode slow = head;
	boolean flag = false;
    
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (fast == slow) {
            flag = true;
            break;
        }
    }
    
    if (flag) {
        fast = head;
        while (fast != slow) {
        	fast = fast.next;
            slow = slow.next;
        }
        return fast;
    }
    return null;
}
```



## 5.分割链表

Question：给定一个链表和一个值K，使得所有小于K的节点在大于等于K的节点的前面。

思路：可以把链表按照值K拆分成两个链表before和after，并持有这两个链表的头节点。遍历原始链表，把节点放入before或after，结束后合并这两个链表即可。需要注意避免出现环形链表。

```java
public ListNode partition(ListNode head,int k) {
	ListNode before_head = new ListNode(0);
    ListNode before = before_head;
    ListNode after_head = new ListNode(0);
    ListNode after = after_head;
    
    while (head != null) {
        if (head.data < k) {
            before.next = head;
            before = before.next;
        }else {
            after.next = head;
            after = after.next;
        }
        
        head = head.next;
    }
    
    // 避免出现环形链表
    after.next = null;
    before.next = after_head.next;
    return before_head.next;
}
```



## 6.反转链表

Question：给定一个链表，请将链表反转过来。例如链表1 -> 2 -> 3 -> 4，反转后4 -> 3 -> 2 -> 1

思路：遍历链表时将前一个元素存储起来。需要注意引用的更改，不会造成意想不到的错误。

```java
public static ListNode resive(ListNode head){
    ListNode prev = null;
    ListNode curr = head;
    while (curr != null) {
       ListNode tempNode = curr.next;
       curr.next = prev;
       prev = curr;
       curr = tempNode;
    }
    return prev;
}
```

