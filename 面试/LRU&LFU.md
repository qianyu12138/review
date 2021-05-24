# LRU&LFU

```java
package com.company;

import java.util.HashMap;
import java.util.LinkedList;
import java.util.Map;
import java.util.Queue;

/**
 * @author 史云龙
 * @version 1.0
 * @title
 * @description
 * @created 2021/1/28 4:58 PM
 * @changeRecord
 */
public class LRUCache {

    /**
     * head -> prev -> node -> next -> tail
     **/
    private Map<Integer,Node> nodeMap = new HashMap<>();
    private class Node{
        int key;
        int value;
        Node prev;
        Node next;

        public Node(int key, int value) {
            this.key = key;
            this.value = value;
        }
    }
    private final Node head = new Node(-1,-1);
    private final Node tail = new Node(-1,-1);

    private int capacity;
    private int actualCapacity;

    public LRUCache(int capacity) {
        head.next = tail;
        tail.prev = head;
        this.capacity = capacity;
    }

    public int get(int key) {
        Node node = nodeMap.get(key);
        if(node == null) return -1;

        moveToHead(node);

//        out();
        return node.value;
    }

    private void moveToHead(Node node) {

        Node oldHead = head.next;

        if(node == oldHead) return;

        node.prev.next = node.next;
        node.next.prev = node.prev;

        head.next = node;
        node.prev = head;
        node.next = oldHead;
        oldHead.prev = node;
    }

    public void put(int key, int value) {

        Node node = nodeMap.get(key);

        if (node == null) {
            if(actualCapacity == capacity){
                if(tail.prev != head) nodeMap.remove(tail.prev.key);
                removeLastNode();
            } else {
                actualCapacity++;
            }
            Node newNode = new Node(key,value);
            addToHead(newNode);
            nodeMap.put(key,newNode);
        } else {
            node.value = value;
            moveToHead(node);
        }
//        out();
    }

    private void addToHead(Node node) {
        Node oldHead = head.next;

        node.next = oldHead;
        oldHead.prev = node;
        head.next = node;
        node.prev = head;
    }

    private void removeLastNode() {
        Node newTail = tail.prev.prev;
        newTail.next = tail;
        tail.prev = newTail;
    }

    private void out(){
        Node n = head;
        while(n != null){
            System.out.print(String.format("[%s-%s]",n.key,n.value));
            n = n.next;
        }
        n = tail;
        while(n != null){
            System.out.print(String.format("[%s-%s]",n.key,n.value));
            n = n.prev;
        }
        System.out.println();
    }
}
```



```java
package com.company;

import java.util.HashMap;
import java.util.Map;
import java.util.TreeMap;

/**
 * @author 史云龙
 * @version 1.0
 * @title
 * @description
 * @created 2021/5/18 5:48 PM
 * @changeRecord
 */
public class LFUCache {

    /**
     * head -> prev -> node -> next -> tail
     **/
    private Map<Integer, Node> nodeMap = new HashMap<>();
    private Map<Integer, Integer> cntMap = new TreeMap<>();

    private class Node {
        int key;
        int value;
        int cnt;
        Node prev;
        Node next;

        public Node(int key, int value, int cnt) {
            this.key = key;
            this.value = value;
            this.cnt = cnt;
        }
    }

    private final Node head = new Node(-1, -1, Integer.MAX_VALUE);
    private final Node tail = new Node(-1, -1, 0);

    private int capacity;
    private int actualCapacity;

    public LFUCache(int capacity) {
        head.next = tail;
        tail.prev = head;
        this.capacity = capacity;
        this.actualCapacity = 0;
    }

    public int get(int key) {
        Node node = nodeMap.get(key);
        if (node == null) {return -1;}

        node.cnt++;
        if(node.cnt ==Integer.MAX_VALUE) throw new RuntimeException();
        moveToNewIndex(node);
        print();
        return node.value;
    }

    //cnt not null
    private void moveToNewIndex(Node node) {
        System.out.println(node.key + "-" + node.value + "-" +node.cnt);
        Node n = tail;
        if(node.prev != null && node.next != null) {
            //本已存在链表中：断链
            node.prev.next = node.next;
            node.next.prev = node.prev;
            n = node.next;
        }

        while(n != head){
            if(n.prev.cnt > node.cnt && n.cnt <= node.cnt) break;
            n = n.prev;
        }
        Node nPrev = n.prev;
        n.prev = node;
        node.next = n;
        nPrev.next = node;
        node.prev = nPrev;
    }

    public void put(int key, int value) {
        if(capacity == 0) return;
        Node node = nodeMap.get(key);
        if(node != null){
            node.value = value;
            node.cnt++;
            moveToNewIndex(node);
        } else {
            //先进行容量判定，因为新节点必须进来，否则进来后cnt=1很容易直接被移除
            actualCapacity++;
            if(actualCapacity>capacity){
                actualCapacity--;
                removeTail();
            }
            Node newNode = new Node(key,value,1);
            moveToNewIndex(newNode);
            nodeMap.put(key,newNode);
        }
        print();
    }

    private void removeTail() {
        Node oldTail = tail.prev;
        nodeMap.remove(oldTail.key);
        oldTail.next.prev = oldTail.prev;
        oldTail.prev.next = oldTail.next;
    }

    private void print(){
        Node n = head;
        while(n != null){

            System.out.print("["+n.key+"-"+n.cnt+"]" + " ");
            n = n.next;
        }
        System.out.println();
    }
}
```