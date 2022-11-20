---
title: 'Leetcode_simple'
tags:
date: '2022-11-20'
categories:
draft: false
---

## 判断是否平衡二叉树
```
func isBalanced(root *TreeNode) bool {
    if root == nil{
        return  true
    }
    return abs(depth(root.Left)-depth(root.Right)) <= 1 && isBalanced(root.Left) && isBalanced(root.Right)
}
```
## 获取树的高度
```
func depth(root *TreeNode) int {
    if root == nil{
        return 0
    }
    return max(depth(root.Left),depth(root.Right)) + 1
}
```
## 二叉树的镜像
```
func mirrorTree(root *TreeNode) *TreeNode {
    if root == nil{
        return nil
    }
    r := mirrorTree(root.Right)
    l := mirrorTree(root.Left)
    root.Right = l
    root.Left = r
    return root
}
```
## 对称的二叉树
```
func isSymmetric(root *TreeNode) bool {
    return checkSymmetric(root,root)## 
```
func checkSymmetric(q,p *TreeNode) bool{
    if q == nil && p == nil{
        return true
    }
    if q == nil || p == nil{
        return false
    }
    return q.Val == p.Val && checkSymmetric(q.Left,p.Right) && checkSymmetric(q.Right,p.Left)
}
```
## 展平二叉搜索树
```
func increasingBST(root *TreeNode) *TreeNode {
    if root == nil{
        return nil
    }
    dummy := &TreeNode{}
    res := dummy

    var dfs func(*TreeNode)
    dfs = func(r *TreeNode){
        if r == nil{
            return 
        }
        dfs(r.Left)
        res.Right = r
        r.Left = nil
        res = r

        dfs(r.Right)
    }
    dfs(root)
    return dummy.Right
}
```
## 二叉树的公共祖先
```
func lowestCommonAncestor(root, p, q *TreeNode) *TreeNode {
    if root == nil{
        return nil
    }

    if root.Val == p.Val || root.Val == q.Val{
        return root
    }
    l,r := lowestCommonAncestor(root.Left,p,q),lowestCommonAncestor(root.Right,p,q)
    if r != nil && l != nil{
        return root
    }  
    if r == nil{
        return l
    }
    return r
}
```
## 是否二叉搜索树
```
func isValidBST(root *TreeNode) bool {
    return helper(root, math.MinInt64, math.MaxInt64)## 
```
func helper(root *TreeNode, lower, upper int) bool {
    if root == nil {
        return true
    }
    if root.Val <= lower || root.Val >= upper {
        return false
    }
    return helper(root.Left, lower, root.Val) && helper(root.Right, root.Val, upper)
}
```

## 从上到下 左到右 分层 打印二叉树, 返回二维数组
```
func levelOrder(root *TreeNode) [][]int {
    ret := [][]int{}
    if root == nil {
        return ret
    }
    q := []*TreeNode{root}
    for i := 0; len(q) > 0; i++ {
        ret = append(ret, []int{})
        p := []*TreeNode{}
        for j := 0; j < len(q); j++ {
            node := q[j]
            ret[i] = append(ret[i], node.Val)
            if node.Left != nil {
                p = append(p, node.Left)
            }
            if node.Right != nil {
                p = append(p, node.Right)
            }
        }
        q = p
    }
    return ret
}
```
## 从上到下 打印二叉树 二
```
func levelOrder(root *TreeNode) []int {
    ret := []int{}
    if root == nil {
        return ret
    }
    q := []*TreeNode{root}
    for i := 0; len(q) > 0; i++ {
        p := []*TreeNode{}
        for j := 0; j < len(q); j++ {
            node := q[j]
            ret = append(ret, node.Val)
            if node.Left != nil {
                p = append(p, node.Left)
            }
            if node.Right != nil {
                p = append(p, node.Right)
            }
        }
        q = p
    }
    return ret
}
```

## 从上到下 奇偶 打印二叉树 三
```
func zigzagLevelOrder(root *TreeNode) (ans [][]int) {
    if root == nil {
        return
    }
    queue := []*TreeNode{root}
    for level := 0; len(queue) > 0; level++ {
        vals := []int{}
        q := queue
        queue = nil
        for _, node := range q {
            vals = append(vals, node.Val)
            if node.Left != nil {
                queue = append(queue, node.Left)
            }
            if node.Right != nil {
                queue = append(queue, node.Right)
            }
        }
        // 本质上和层序遍历一样，我们只需要把奇数层的元素翻转即可
        if level%2 == 1 {
            for i, n := 0, len(vals); i < n/2; i++ {
                vals[i], vals[n-1-i] = vals[n-1-i], vals[i]
            }
        }
        ans = append(ans, vals)
    }
    return
}
```
## 二叉树的第 k 大节点
```
func kthLargest(root *TreeNode, k int) int {
    var dfs func(*TreeNode)
    var res = -1
    dfs = func(node *TreeNode){
        if node == nil{
            return 
        }
        dfs(node.Right)
        k--
        if k == 0 {
            res = node.Val
            return 
        }
        dfs(node.Left)
    }
    dfs(root)
    return res
}
```
## 二叉搜索树中两个节点之和
```
func findTarget(root *TreeNode, k int) bool {
    set := map[int]struct{}{}
    var dfs func(*TreeNode) bool
    dfs = func(node *TreeNode) bool {
        if node == nil {
            return false
        }
        if _, ok := set[k-node.Val]; ok {
            return true
        }
        set[node.Val] = struct{}{}
        return dfs(node.Left) || dfs(node.Right)
    }
    return dfs(root)
}
```

## 二叉树的直径
```
func diameterOfBinaryTree(root *TreeNode) int{
    var ans = 1
    var depth func(*TreeNode)int
    depth = func(node *TreeNode)int{
        if node == nil{
            return 0
        }
        l, r := depth(node.Left), depth(node.Right)
        ans = max(ans, l + r + 1)
        return max(l, r) + 1
    }
    depth(root)
    return ans - 1
}
```
## 二叉树最深最左
```
func findBottomLeftValue(root *TreeNode) (ans int) {
    q := []*TreeNode{root}
    for len(q) > 0 {
        node := q[0]
        q = q[1:]
        if node.Right != nil {
            q = append(q, node.Right)
        }
        if node.Left != nil {
            q = append(q, node.Left)
        }
        ans = node.Val
    }
    return
}

func findBottomLeftValue(root *TreeNode) (curVal int) {
    curHeight := 0
    var dfs func(*TreeNode, int)
    dfs = func(node *TreeNode, height int) {
        if node == nil {
            return
        }
        height++
        dfs(node.Left, height)
        dfs(node.Right, height)
        if height > curHeight {
            curHeight = height
            curVal = node.Val
        }
    }
    dfs(root, 0)
    return
}
```

## 合并二叉树
```
func mergeTrees(t1, t2 *TreeNode) *TreeNode {
    if t1 == nil {
        return t2
    }
    if t2 == nil {
        return t1
    }
    t1.Val += t2.Val
    t1.Left = mergeTrees(t1.Left, t2.Left)
    t1.Right = mergeTrees(t1.Right, t2.Right)
    return t1
}
```
## 二叉树的坡度
```
func findTilt(root *TreeNode) (ans int) {
    var dfs func(*TreeNode) int
    dfs = func(node *TreeNode) int {
        if node == nil {
            return 0
        }
        sumLeft := dfs(node.Left)
        sumRight := dfs(node.Right)
        ans += abs(sumLeft - sumRight)
        return sumLeft + sumRight + node.Val
    }
    dfs(root)
    return
}
```

## 单值二叉树
```
func isUnivalTree(root *TreeNode) bool {
    return root == nil || (root.Left == nil || root.Val == root.Left.Val && isUnivalTree(root.Left)) &&
                         (root.Right == nil || root.Val == root.Right.Val && isUnivalTree(root.Right))
}
```
## 单值二叉树
```
func isUnivalTree(root *TreeNode) bool {
    if root == nil{
        return false
    }
    var dfs func(root *TreeNode) bool
    dfs = func(node *TreeNode) bool {
        if node == nil {
            return true
        }
        if node.Val != root.Val{
            return false
        }
        return dfs(node.Left) && dfs(node.Right)
    }
    
    return dfs(root)
}
```
## 重建二叉树
```
func buildTree(preorder []int, inorder []int) *TreeNode {
    if len(preorder) == 0 {
        return nil
    }
    root := &TreeNode{preorder[0], nil, nil}
    i := 0
    for ; i < len(inorder); i++ {
        if inorder[i] == preorder[0] {
            break
        }
    }
    root.Left = buildTree(preorder[1:len(inorder[:i])+1], inorder[:i])
    root.Right = buildTree(preorder[len(inorder[:i])+1:], inorder[i+1:])
    return root
}
```

## 二叉搜索树，数组是否是后序
```
class Solution:
    def verifyPostorder(self, postorder: [int]) -> bool:
        def recur(i, j):
            if i >= j: return True
            p = i
            while postorder[p] < postorder[j]: p += 1
            m = p
            while postorder[p] > postorder[j]: p += 1
            return p == j and recur(i, m - 1) and recur(m, j - 1)

        return recur(0, len(postorder) - 1## 
```

## 二叉树中 和 为某值 的路径
```
func pathSum(root *TreeNode, target int) (ans [][]int) {
    path := []int{}
    var dfs func(*TreeNode, int)
    dfs = func(node *TreeNode, left int) {
        if node == nil {
            return
        }
        left -= node.Val
        path = append(path, node.Val)
        defer func() { path = path[:len(path)-1] }()
        if node.Left == nil && node.Right == nil && left == 0 {
            ans = append(ans, append([]int(nil), path...))
            return
        }
        dfs(node.Left, left)
        dfs(node.Right, left)
    }
    dfs(root, target)
    return
}
```

## 没有重复项的字符串全排列
```
func permutation(s string) (ans []string) {
    t := []byte(s)
    sort.Slice(t, func(i, j int) bool { return t[i] < t[j] })
    n := len(t)
    perm := make([]byte, 0, n)
    vis := make([]bool, n)
    var backtrack func(int)
    backtrack = func(i int) {
        if i == n {
            ans = append(ans, string(perm))
            return
        }
        for j, b := range vis {
            if b || j > 0 && !vis[j-1] && t[j-1] == t[j] {
                continue
            }
            vis[j] = true
            perm = append(perm, t[j])
            backtrack(i + 1)
            perm = perm[:len(perm)-1]
            vis[j] = false
        }
    }
    backtrack(0)
    return
}
```


## 反转链表
```
func reverseList(head *ListNode) *ListNode {
    var prev *ListNode
    curr := head
    for curr != nil {
        next := curr.Next // b->c c nil
        curr.Next = prev // a -> nil, b -> a, c -> b -> a
        prev = curr // a -> nil, b->a
        curr = next // b->c, c
    }
    return prev
}
```
## 反转链表
```
func reverseList(head *ListNode) *ListNode {
    if head == nil || head.Next == nil {
        return head
    }
    newHead := reverseList(head.Next)
    head.Next.Next = head
    head.Next = nil
    return newHead
}
```
## 倒序打印链表 递归 或者 使用计算链表长度，然后数组倒序放入
```
func reversePrint(head *ListNode) (out []int) {
 if head == nil {
        return []int{}
    }
    return append(reversePrint(head.Next),head.Val)
}
```
## 合并两个有序链表
```
func mergeTwoLists(l1 *ListNode, l2 *ListNode) *ListNode {
    dummy := &ListNode{}
    node := dummy
    for l1 != nil && l2 != nil{
        if l1.Val <= l2.Val{
            node.Next = l1
            l1 = l1.Next
        }else{
            node.Next = l2
            l2 = l2.Next
        }
        node = node.Next
    }
    if l1 != nil{
        node.Next = l1
    }else{
        node.Next = l2
    }
    return dummy.Next
}
```
## 链表的公共节点
```
func getIntersectionNode(headA, headB *ListNode) *ListNode {
    if headA == nil || headB == nil {
        return nil
    }
    pa, pb := headA, headB
    for pa != pb {
        if pa == nil {
            pa = headB
        } else {
            pa = pa.Next
        }
        if pb == nil {
            pb = headA
        } else {
            pb = pb.Next
        }
    }
    return pa
}
```
## 环形链表
```
func hasCycle(head *ListNode) bool {
    if head == nil || head.Next == nil {
        return false
    }
    slow, fast := head, head.Next
    for fast != slow {
        if fast == nil || fast.Next == nil {
            return false
        }
        slow = slow.Next
        fast = fast.Next.Next
    }
    return true
}
```

## 1的个数
```
func hammingWeight(num uint32) (ones int) {
    for ; num > 0; num &= num - 1 {
        ones++
    }
    return
}
```
// 1,2,3,4,5

## 旋转数组中的最小值
```
func minArray(numbers []int) int {
    low := 0
    high := len(numbers) - 1
    for low < high {
        pivot := low + (high - low) / 2
        if numbers[pivot] < numbers[high] {
            high = pivot
        } else if numbers[pivot] > numbers[high] {
            low = pivot + 1
        } else {
            high--
        }
    }
    return numbers[low]
}
```
## 连续子数组的最大和
```
func maxSubArray(nums []int) int {
    max := nums[0]
    for i := 1; i < len(nums); i++ {
        if nums[i-1] > 0 {
            nums[i] += nums[i-1]
        }
        if nums[i] > max {
            max = nums[i]
        }
    }
    return max
}
```
## 股票最佳时机2
```
func maxProfit(prices []int) (ans int) {
    for i := 1; i < len(prices); i++ {
        ans += max(0, prices[i]-prices[i-1])
    }
    return
}
```
## 股票最佳时机1
```
func maxProfit(prices []int) (ans int) {
    min := math.MaxInt64
    max := 0
    for _,v := range prices{
        if v < min{
            min = v
        }
        if v - min > max{
            max = v-min
        }
    }
    return max
}
```

## 杨辉三角
```
func generate(numRows int) [][]int {
    ans := make([][]int, numRows)
    for i := range ans {
        ans[i] = make([]int, i+1)
        ans[i][0] = 1
        ans[i][i] = 1
        for j := 1; j < i; j++ {
            ans[i][j] = ans[i-1][j] + ans[i-1][j-1]
        }
    }
    return ans
}
```
## 杨辉三角 第 n 行
```
func getRow(rowIndex int) []int {
    row := make([]int, rowIndex+1)
    row[0] = 1
    for i := 1; i <= rowIndex; i++ {
        for j := i; j > 0; j-- {
            row[j] += row[j-1]
        }
    }
    return row
}
```
## 杨辉三角 第 n 行
```
func getRow(rowIndex int) []int {
    row := make([]int, rowIndex+1)
    row[0] = 1
    for i := 1; i <= rowIndex; i++ {
        row[i] = row[i-1] * (rowIndex - i + 1) / i
    }
    return row
}
```

## 其他

```
func max(a,b int) int{
    if a>b{
        return a
    }
    return b## 
}
func abs(a int) int{
    if a<0{
        return -a
    }
    return a
}
```