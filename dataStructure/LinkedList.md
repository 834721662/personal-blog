[TOC]


source code:http://hg.openjdk.java.net/jdk8/jdk8/jdk/file/687fd7c7986d/src/share/classes/java/util/LinkedList.java


```java
package java.util;    

public class LinkedList<E>    
    extends AbstractSequentialList<E>    
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable    
{    
    // 链表的表头
    transient Node<E> first;

    // 链表的尾巴
    transient Node<E> last;

    // LinkedList中元素个数    
    transient int size = 0;    

    // 默认构造函数：创建一个空的链表    
    public LinkedList() {
    }

    // 包含“集合”的构造函数:创建一个包含“集合”的LinkedList    
    public LinkedList(Collection<? extends E> c) {    
        this();    
        addAll(c);    
    }    


    // 把e设置为第一个元素，如果第一个元素为空，则当作第一个元素(同时也是最后一个元素)
    private void linkFirst(E e) {
        final Node<E> f = first;
        final Node<E> newNode = new Node<>(null, e, f);  // 前驱为null，元素为e，后置为f
        first = newNode;
        if (f == null)
            last = newNode;
        else
            f.prev = newNode;
        size++;
        modCount++;
    }

    // 注意这里的修饰符是空的 即 default
    // 把e设置为最后一个元素，如果最后一个元素为空 -> 同上
    void linkLast(E e) {
        final Node<E> l = last;
        final Node<E> newNode = new Node<>(l, e, null);
        last = newNode;
        if (l == null)
            first = newNode;
        else
            l.next = newNode;
        size++;
        modCount++;
    }

    // 将元素e插入到非空元素succ之前
    void linkBefore(E e, Node<E> succ) {
        // assert succ != null;
        final Node<E> pred = succ.prev;
        final Node<E> newNode = new Node<>(pred, e, succ); 
        succ.prev = newNode;
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        size++;
        modCount++;
    }

    // 取消非空节点f的链接
    private E unlinkFirst(Node<E> f) {
        // assert f == first && f != null;
        final E element = f.item;
        final Node<E> next = f.next;
        f.item = null;
        f.next = null; // help GC
        first = next;
        if (next == null)
            last = null;
        else
            next.prev = null;
        size--;
        modCount++;
        return element;
    }

    // 取消最后一个节点l的链接
	private E unlinkLast(Node<E> l) {
        // assert l == last && l != null;
        final E element = l.item;
        final Node<E> prev = l.prev;
        l.item = null;
        l.prev = null; // help GC
        last = prev;
        if (prev == null)
            first = null;
        else
            prev.next = null;
        size--;
        modCount++;
        return element;
    }


    E unlink(Node<E> x) {   //上面是找元素，这个方法是真正删除元素
        final E element = x.item;   //x.item表示当前的x节点
        final Node<E> next = x.next;    //x.next表示x后继引用，next同
        final Node<E> prev = x.prev;    //x.prev是x的前继引用，prev同
        ......

        if (prev == null) { //如果prev为null，则表示x为第一个结点，此时删除x的做法只需要将x的
            first = next;   //下一个节点设为第一个节点即可，first表示链表的第一节点。
        } else {
     ①      prev.next = next;   //否则的话，x为普通节点。那么只要将x的前一个结点(prev)的后继引用指向x的下一个 
            x.prev = null;      //节点就行了,也就是(向后)跳过了x节点。x的前继引用删除，断开与前面元素的联系。
        }

        if (next == null) {
            last = prev;    //如果x节点后继无人，说明他是最后一个节点。直接把前一个结点作为链表的最后一个节点就行
        } else {
     ②      next.prev = prev;   //否则x为普通节点，将x的下一个节点的前继引用指向x的前一个节点，也就是(向前)跳过x.   
            x.next = null;  //x的后继引用删除，断了x的与后面元素的联系
        }

        x.item = null;  //删除x自身
        size--;     //链表长度减1
        modCount++;
        return element;
    }

    // 获取LinkedList的第一个元素    
    public E getFirst() {    
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return f.item;  
    }    

    // 获取LinkedList的最后一个元素    
    public E getLast() {
        final Node<E> l = last;
        if (l == null)
            throw new NoSuchElementException();
        return l.item;
    }

    // 删除LinkedList的第一个元素    
    public E removeFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return unlinkFirst(f);
    }

    // 删除LinkedList的最后一个元素    
    public E removeLast() {
        final Node<E> l = last;
        if (l == null)
            throw new NoSuchElementException();
        return unlinkLast(l);
    }

    // 将元素添加到LinkedList的起始位置    
    public void addFirst(E e) {
        linkFirst(e);
    }

    // 将元素添加到LinkedList的结束位置    
    public void addLast(E e) {
        linkLast(e);
    } 

    // 判断LinkedList是否包含元素(o)    
    public boolean contains(Object o) {
        return indexOf(o) != -1;
    }

    // 返回LinkedList的大小    
    public int size() {    
        return size;    
    }    

    // 将元素(E)添加到LinkedList中    
    public boolean add(E e) {
    	// 默认添加到最后一位
        linkLast(e);
        return true;
    } 

    // 从LinkedList中删除元素(o)    
    // 从链表开始查找，如存在元素(o)则删除该元素并返回true；    
    // 否则，返回false。    
    public boolean remove(Object o) {    
        if (o == null) { // 如果要删除的这个元素为 null
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null) {
                    unlink(x);
                    return true;
                }
            }
        } else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item)) {
                    unlink(x);
                    return true;
                }
            }
        }
        return false;    
    }    

    // 将“集合(c)”添加到LinkedList中。    
    // 实际上，是从双向链表的末尾开始，将“集合(c)”添加到双向链表中。    
    public boolean addAll(Collection<? extends E> c) {    
        return addAll(size, c);    
    }    

    // 从双向链表的index开始，将“集合(c)”添加到双向链表中。    
    public boolean addAll(int index, Collection<? extends E> c) {    
        checkPositionIndex(index); // index时候超过size

        Object[] a = c.toArray();    
        // 获取集合的长度    
        int numNew = a.length;    
        if (numNew==0)    
            return false;    
        modCount++;    

         Node<E> pred, succ;
        if (index == size) { // 最后一个位置
            succ = null; 
            pred = last;
        } else {
            succ = node(index);  // succ 指向index节点
            pred = succ.prev;  
        }

       // 将集合c全部加入双向链表当中
       for (Object o : a) {
            @SuppressWarnings("unchecked") E e = (E) o;
            Node<E> newNode = new Node<>(pred, e, null); // 前驱为 pred
            if (pred == null)
                first = newNode;
            else
                pred.next = newNode;
            pred = newNode;
        }

        if (succ == null) {
            last = pred;
        } else {
            pred.next = succ;
            succ.prev = pred;
        }  

        // 调整LinkedList的实际大小    
        size += numNew;    
        modCount++;
        return true;    
    }    

    // 清空双向链表    
    public void clear() {    
        // 从表头开始，逐个向后遍历；对遍历到的节点执行一下操作：    
        // (01) 设置前一个节点为null     
        // (02) 设置当前节点的内容为null     
        // (03) 设置后一个节点为“新的当前节点”    
        for (Node<E> x = first; x != null; ) {
            Node<E> next = x.next;
            x.item = null;
            x.next = null;
            x.prev = null;
            x = next;
        }    
        first = last = null;
        // 设置大小为0    
        size = 0;    
        modCount++;    
    }    

    // 返回LinkedList指定位置的元素    
    public E get(int index) {
        checkElementIndex(index);
        return node(index).item;
    }

    // 设置index位置对应的节点的值为element    
    public E set(int index, E element) {
        checkElementIndex(index);
        Node<E> x = node(index);
        E oldVal = x.item;
        x.item = element;
        return oldVal;
    }

    // 在index前添加节点，且节点的值为element    
    public void add(int index, E element) {
        checkPositionIndex(index);

        if (index == size)
            linkLast(element);
        else
            linkBefore(element, node(index));
    }

    // 删除index位置的节点    
    public E remove(int index) {
        checkElementIndex(index);
        return unlink(node(index));
    }

    // 判断index是否是符合要求
    private boolean isElementIndex(int index) {
        return index >= 0 && index < size;
    }

    // 判断索引index是否是一个有效的位置
    private boolean isPositionIndex(int index) {
        return index >= 0 && index <= size;
    }

    // 获取双向链表中指定位置的节点    
    Node<E> node(int index) {
        // assert isElementIndex(index);

        // 获取index处的节点。    
        // 若index < 双向链表长度的1/2,则从前先后查找;    
        // 否则，从后向前查找。   
        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }   

    // 从前向后查找，返回“值为对象(o)的节点对应的索引”    
    // 不存在就返回-1    
    public int indexOf(Object o) {    
        int index = 0;    
        if (o==null) {    
            for (Node e = prev.next; e != prev; e = e.next) {    
                if (e.item==null)    
                    return index;    
                index++;    
            }    
        } else {    
            for (Node e = prev.next; e != prev; e = e.next) {    
                if (o.equals(e.item))    
                    return index;    
                index++;    
            }    
        }    
        return -1;    
    }    

    // 从后向前查找，返回“值为对象(o)的节点对应的索引”    
    // 不存在就返回-1    
    public int lastIndexOf(Object o) {    
        int index = size;
        if (o == null) {
            for (Node<E> x = last; x != null; x = x.prev) {
                index--;
                if (x.item == null)
                    return index;
            }
        } else {
            for (Node<E> x = last; x != null; x = x.prev) {
                index--;
                if (o.equals(x.item))
                    return index;
            }
        }
        return -1;
    }    

    // 返回第一个节点    
    // 若LinkedList的大小为0,则返回null    
    public E peek() {    
        final Node<E> f = first;
        return (f == null) ? null : f.item;
    }    

    // 返回第一个节点    
    // 若LinkedList的大小为0,则抛出异常    
    public E element() {    
        return getFirst();    
    }    

    // 删除并返回第一个节点    
    // 若LinkedList的大小为0,则返回null    
    public E poll() {    
        final Node<E> f = first;
        return (f == null) ? null : unlinkFirst(f);
    }    

    // 删除第一个元素
    public E remove() {
        return removeFirst();
    }


    // 将e添加双向链表末尾    
    public boolean offer(E e) {    
        return add(e);    
    }    

    // 将e添加双向链表开头    
    public boolean offerFirst(E e) {    
        addFirst(e);    
        return true;    
    }    

    // 将e添加双向链表末尾    
    public boolean offerLast(E e) {    
        addLast(e);    
        return true;    
    }    

    // 返回第一个节点    
    // 若LinkedList的大小为0,则返回null    
    public E peekFirst() {    
        final Node<E> f = first;
        return (f == null) ? null : f.item;
    }    

    // 返回最后一个节点    
    // 若LinkedList的大小为0,则返回null    
    public E peekLast() {    
        final Node<E> l = last;
        return (l == null) ? null : l.item;   
    }    

    // 删除并返回第一个节点    
    // 若LinkedList的大小为0,则返回null    
    public E pollFirst() {    
        final Node<E> f = first;
        return (f == null) ? null : unlinkFirst(f);
    }    

    // 删除并返回最后一个节点    
    // 若LinkedList的大小为0,则返回null    
    public E pollLast() {    
         final Node<E> l = last;
        return (l == null) ? null : unlinkLast(l);   
    }    

    // 将e插入到双向链表开头    
    public void push(E e) {    
        addFirst(e);    
    }    

    // 删除并返回第一个节点    
    public E pop() {    
        return removeFirst();    
    }    

    // 从LinkedList开始向后查找，删除第一个值为元素(o)的节点    
    // 从链表开始查找，如存在节点的值为元素(o)的节点，则删除该节点    
    public boolean removeFirstOccurrence(Object o) {    
        return remove(o);    
    }    

    // 从LinkedList末尾向前查找，删除第一个值为元素(o)的节点    
    // 从链表开始查找，如存在节点的值为元素(o)的节点，则删除该节点    
    public boolean removeLastOccurrence(Object o) {    
        if (o==null) {    
            for (Node<E> e = prev.previous; e != prev; e = e.previous) {    
                if (e.item==null) {    
                    remove(e);    
                    return true;    
                }    
            }    
        } else {    
            for (Node<E> e = prev.previous; e != prev; e = e.previous) {    
                if (o.equals(e.item)) {    
                    remove(e);    
                    return true;    
                }    
            }    
        }    
        return false;    
    }    

    // 返回“index到末尾的全部节点”对应的ListIterator对象(List迭代器)    
    public ListIterator<E> listIterator(int index) {    
    	checkPositionIndex(index);
        return new ListItr(index);    
    }    

    // List迭代器    
    private class ListItr implements ListIterator<E> {    
        // 上一次返回的节点    
        private Node<E> lastReturned = null;
        // 下一个节点    
        private Node<E> next;
        // 下一个节点对应的索引值    
        private int nextIndex;
        // 期望的改变计数。用来实现fail-fast机制。    
        private int expectedModCount = modCount;    

        // 构造函数。    
        // 从index位置开始进行迭代    
        ListItr(int index) {    
            next = (index == size) ? null : node(index);
            nextIndex = index; 
        }    

        // 是否存在下一个元素    
        public boolean hasNext() {    
            // 通过元素索引是否小于“双向链表大小”来判断是否达到最后。    
            return nextIndex < size;    
        }    

        // 获取下一个元素    
        public E next() {    
            checkForComodification();    
            if (!hasNext())
                throw new NoSuchElementException();    

            lastReturned = next;    
            // next指向链表的下一个元素    
            next = next.next;    
            nextIndex++;    
            return lastReturned.item;    
        }    

        // 是否存在上一个元素    
        public boolean hasPrevious() {    
            // 通过元素索引是否大于0，来判断是否达到开头。    
            return nextIndex > 0;    
        }    

        // 获取上一个元素    
        public E previous() {    
            if (nextIndex == 0)    
            throw new NoSuchElementException();    

            // next指向链表的上一个元素    
            lastReturned = next = next.previous;    
            nextIndex--;    
            checkForComodification();    
            return lastReturned.element;    
        }    

        // 获取下一个元素的索引    
        public int nextIndex() {    
            return nextIndex;    
        }    

        // 获取上一个元素的索引    
        public int previousIndex() {    
            return nextIndex-1;    
        }    

        // 删除当前元素。    
        // 删除双向链表中的当前节点    
        public void remove() {    
            checkForComodification();    
            Node<E> lastNext = lastReturned.next;    
            try {    
                LinkedList.this.remove(lastReturned);    
            } catch (NoSuchElementException e) {    
                throw new IllegalStateException();    
            }    
            if (next==lastReturned)    
                next = lastNext;    
            else   
                nextIndex--;    
            lastReturned = prev;    
            expectedModCount++;    
        }    

        // 设置当前节点为e    
        public void set(E e) {    
            if (lastReturned == prev)    
                throw new IllegalStateException();    
            checkForComodification();    
            lastReturned.element = e;    
        }    

        // 将e添加到当前节点的前面    
        public void add(E e) {    
            checkForComodification();    
            lastReturned = prev;    
            addBefore(e, next);    
            nextIndex++;    
            expectedModCount++;    
        }    

        // 判断 “modCount和expectedModCount是否相等”，依次来实现fail-fast机制。    
        final void checkForComodification() {    
            if (modCount != expectedModCount)    
            throw new ConcurrentModificationException();    
        }    
    }    

    // 双向链表的节点所对应的数据结构。    
    // 包含3部分：上一节点，下一节点，当前节点值。    
    private static class Node<E> {    
        // 当前节点所包含的值    
        E item;    
        // 下一个节点    
        Node<E> next;    
        // 上一个节点    
        Node<E> previous;    

        /**   
         * 链表节点的构造函数。   
         * 参数说明：   
         *   item  —— 节点所包含的数据   
         *   next      —— 下一个节点   
         *   previous —— 上一个节点   
         */   
        Node(E element, Node<E> next, Node<E> previous) {    
            this.item = item;    
            this.next = next;    
            this.previous = previous;    
        }    
    }    

    // 将节点(节点数据是e)添加到Node节点之前。    
    private Node<E> addBefore(E e, Node<E> Node) {    
        // 新建节点newNode，将newNode插入到节点e之前；并且设置newNode的数据是e    
        Node<E> newNode = new Node<E>(e, Node, Node.previous);    
        newNode.previous.next = newNode;    
        newNode.next.previous = newNode;    
        // 修改LinkedList大小    
        size++;    
        // 修改LinkedList的修改统计数：用来实现fail-fast机制。    
        modCount++;    
        return newNode;    
    }    

    // 将节点从链表中删除    
    private E remove(Node<E> e) {    
        if (e == prev)    
            throw new NoSuchElementException();    

        E result = e.item;    
        e.previous.next = e.next;    
        e.next.previous = e.previous;    
        e.next = e.previous = null;    
        e.item = null;    
        size--;    
        modCount++;    
        return result;    
    }    

    // 反向迭代器    
    public Iterator<E> descendingIterator() {    
        return new DescendingIterator();    
    }    

    // 反向迭代器实现类。    
    private class DescendingIterator implements Iterator {    
        final ListItr itr = new ListItr(size());    
        // 反向迭代器是否下一个元素。    
        // 实际上是判断双向链表的当前节点是否达到开头    
        public boolean hasNext() {    
            return itr.hasPrevious();    
        }    
        // 反向迭代器获取下一个元素。    
        // 实际上是获取双向链表的前一个节点    
        public E next() {    
            return itr.previous();    
        }    
        // 删除当前节点    
        public void remove() {    
            itr.remove();    
        }    
    }    


    // 返回LinkedList的Object[]数组    
    public Object[] toArray() {    
    // 新建Object[]数组    
    Object[] result = new Object[size];    
        int i = 0;    
        // 将链表中所有节点的数据都添加到Object[]数组中    
        for (Node<E> e = prev.next; e != prev; e = e.next)    
            result[i++] = e.item;    
    return result;    
    }    

    // 返回LinkedList的模板数组。所谓模板数组，即可以将T设为任意的数据类型    
    public <T> T[] toArray(T[] a) {    
        // 若数组a的大小 < LinkedList的元素个数(意味着数组a不能容纳LinkedList中全部元素)    
        // 则新建一个T[]数组，T[]的大小为LinkedList大小，并将该T[]赋值给a。    
        if (a.length < size)    
            a = (T[])java.lang.reflect.Array.newInstance(    
                                a.getClass().getComponentType(), size);    
        // 将链表中所有节点的数据都添加到数组a中    
        int i = 0;    
        Object[] result = a;    
        for (Node<E> e = prev.next; e != prev; e = e.next)    
            result[i++] = e.item;    

        if (a.length > size)    
            a[size] = null;    

        return a;    
    }    


    // 克隆函数。返回LinkedList的克隆对象。    
    public Object clone() {    
        LinkedList<E> clone = null;    
        // 克隆一个LinkedList克隆对象    
        try {    
            clone = (LinkedList<E>) super.clone();    
        } catch (CloneNotSupportedException e) {    
            throw new InternalError();    
        }    

        // 新建LinkedList表头节点    
        clone.prev = new Node<E>(null, null, null);    
        clone.prev.next = clone.prev.previous = clone.prev;    
        clone.size = 0;    
        clone.modCount = 0;    

        // 将链表中所有节点的数据都添加到克隆对象中    
        for (Node<E> e = prev.next; e != prev; e = e.next)    
            clone.add(e.item);    

        return clone;    
    }    

    // java.io.Serializable的写入函数    
    // 将LinkedList的“容量，所有的元素值”都写入到输出流中    
    private void writeObject(java.io.ObjectOutputStream s)    
        throws java.io.IOException {    
        // Write out any hidden serialization magic    
        s.defaultWriteObject();    

        // 写入“容量”    
        s.writeInt(size);    

        // 将链表中所有节点的数据都写入到输出流中    
        for (Node e = prev.next; e != prev; e = e.next)    
            s.writeObject(e.item);    
    }    

    // java.io.Serializable的读取函数：根据写入方式反向读出    
    // 先将LinkedList的“容量”读出，然后将“所有的元素值”读出    
    private void readObject(java.io.ObjectInputStream s)    
        throws java.io.IOException, ClassNotFoundException {    
        // Read in any hidden serialization magic    
        s.defaultReadObject();    

        // 从输入流中读取“容量”    
        int size = s.readInt();    

        // 新建链表表头节点    
        prev = new Node<E>(null, null, null);    
        prev.next = prev.previous = prev;    

        // 从输入流中将“所有的元素值”并逐个添加到链表中    
        for (int i=0; i<size; i++)    
            addBefore((E)s.readObject(), prev);    
    }    

}   
```

#### 总结

1. LinkedList现在有两个元素: first & last 分别代表链表的头和尾元素，在旧版本的jdk当中，这两个元素是不存东西的，但是新版本当中这两个元素是链表成员。

2. 注意两个不同的构造方法。
无参构造方法啥都不做，包含Collection的构造方法，先调用无参构造方法，然后将Collection中的数据加入到链表的尾部后面。

3. 在查找和删除某元素时，源码中都划分为该元素为null和不为null两种情况来处理，LinkedList中允许元素为null。

4. 基于链表实现的，不存在容量不足的问题。

5. 注意源码中的Node node(int index)方法。
该方法返回双向链表中指定位置处的节点，而链表中是没有下标索引的，要指定位置出的元素，就要遍历该链表，从源码的实现中，我们看到这里有一个加速动作。
源码中先将index与长度size的一半比较，如果index < size / 2，就只从位置0往后遍历到位置index处;
而如果index > size / 2，就只从位置size往前遍历到位置index处。这样可以减少一部分不必要的遍历，从而提高一定的效率（实际上效率还是很低）

6. LinkedList是基于链表实现的，因此插入删除效率高，查找效率低

7. 要注意源码中还实现了栈和队列的操作方法，因此也可以作为栈、队列和双端队列来使用


#### Q & A

ArrayList 和 LinkedList 的区别是啥

(1)顺序插入速度ArrayList会比较快，因为ArrayList是基于数组实现的，数组是事先new好的，只要往指定位置塞一个数据就好了；
LinkedList则不同，每次顺序插入的时候LinkedList将new一个对象出来，如果对象比较大，那么new的时间势必会长一点，再加上一些引用赋值的操作，所以顺序插入LinkedList必然慢于ArrayList
(2)基于上一点，因为LinkedList里面不仅维护了待插入的元素，还维护了Node的前置Node和后继Node，如果一个LinkedList中的Node非常多，那么LinkedList将比ArrayList更耗费一些内存
(3)数据遍历的速度：使用各自遍历效率最高的方式，ArrayList的遍历效率会比LinkedList的遍历效率高一些
(4)有些说法认为LinkedList做插入和删除更快，这种说法其实是不准确的：
  ①LinkedList做插入、删除的时候，慢在寻址，快在只需要改变Node前后引用
  ②ArrayList做插入、删除的时候，慢在数组元素的批量copy，快在寻址
所以，如果待插入、删除的元素是在数据结构的前半段尤其是非常靠前的位置的时候，LinkedList的效率将大大快过ArrayList，因为ArrayList将批量copy大量的元素；
越往后，对于LinkedList来说，因为它是双向链表，所以在第2个元素后面插入一个数据和在倒数第2个元素后面插入一个元素在效率上基本没有差别，但是ArrayList由于要批量copy的元素越来越少，操作速度必然追上乃至超过LinkedList。
从这个分析看出，如果程序确定插入、删除的元素是在前半段，那么就使用LinkedList；如果确定删除、删除的元素在比较靠后的位置，那么可以考虑使用ArrayList。
如果你不能确定你要做的插入、删除是在哪儿呢？还是建议使用LinkedList吧，因为LinkedList整体插入、删除的执行效率比较稳定，没有ArrayList这种越往后越快的情况；
而且插入元素的时候，弄得不好ArrayList就要进行一次扩容，所以记住，ArrayList底层数组扩容是一个既消耗时间又消耗空间的操作。


