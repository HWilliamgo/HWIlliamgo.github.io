此篇文章是对源码的分析，但并没有贴大量源码。
![](https://upload-images.jianshu.io/upload_images/7177220-d0e39ac8fdd373e8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](https://upload-images.jianshu.io/upload_images/7177220-996ee0b36ea10f52.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
3个接口前两个就不用多说了，Map<K,V>接口也应该对着中文版api文档看一下就明白了。

先看直接父类AbstractMap<K,V>：
## AbstractMap<K,V>
### 内部类：
1. `SimpleEntry<K,V>`:实现`Entry`接口，Entry的英文单词是条目的意思，接口里面都是一些对规定好的操作方法，看文档就行了。在类中只有`final key`和`value`两个字段，其中由于`key`是final的，因此初始化之后就不可变。实例方法都是对这两个变量的基本操作，没什么特别的。简单来说就是对`key`和`value`的一个封装。
2. `SimpleImmutableEntry<K,V>`和上述没有太大差异，也是实现`Entry`接口，差异在于`key`和`value`都带有final，此外调用`setValue`会抛出异常。
### 实例变量：
1. `Set<K> keySet;`
2. `Collection<V> values;`
### 关键方法：
1. 
	``` java
	public abstract Set<Entry<K,V>> entrySet();
	```
	它是抽象方法，没有实现。它的返回值是一个类型为Entry的Set的对象，该Set对象中保存了Entry对象。也就是说有了这个方法，才能进行对真正的Entry<K,V>的操作。并且AbstractMap中许多方法都通过这个方法来直接返回Set对象，接着再操作，比如`Iterator<Map.Entry<K,V>> i = entrySet().iterator();`。我的疑问是，为什么不添加一个实例变量来保存这个Entry对象，而要每次都调用一个方法来返回对象。
2. 
	``` java
	public Set<K> keySet(){...}
	```
	创建一个`AbstractSet`类（实现了`Set`接口）的子类的实现并返回。返回的对象的类中重写了`AbstractSet`的所有方法，其中重写的`iterator()`较为重要：
	``` java
	public Iterator<K> iterator() {
		return new Iterator<K>() {
			//将entrySet.iterator的返回值作为当前要返回的迭代器
			//当前对象就可以轻松实现对所有key的访问。
			private Iterator<Entry<K,V>> i = entrySet().iterator();
			
			public boolean hasNext() {return i.hasNext();}
			
			public K next() {return i.next().getKey();}
			
			public void remove() {i.remove();}
		};
	}
	```
	这样就对外界打开了可以访问key的接口，通过这个`keySet()`方法，可以拿到所有的key，但是这里如果对返回的`Set`中的key做出修改，那么也会影响整个`Map`中`Entry<K,V>`的key，因为返回的Set中的元素仍然指向原有`Entry<K,V>`中的引用。
3. 
	``` java
	public Collection<V> values(){...}
	```
	上面的方法实现类似，直接拿entrySet().iterator()的返回值作为AbstractCollection子类的iterator方法的返回值，作用是类似的。
	
	
---

## JDK1.7 的HashMap
### 内部类：
1. `private static Holder`:保存了3个运行时才可确定的静态常量。
2. `private static Entry<K,V>`:实现了`Map.Eentry<K,V>`接口，4个实例变量：
	``` java
	final K key;
	V value;
	Entry<K,V> next;//通过这个引用使Entry具有了单链表的属性
	int hash;
	```
	重写了equals，只要两个对象的key和value的值都equals，就返回true。
3. `private abstract class HashIterator<E> implements Iterator<E>`
是一个抽象类 ，直接看下源码。
``` java
private abstract class HashIterator<E> implements Iterator<E> {
	Entry<K,V> next;        // next entry to return
	int expectedModCount;   // For fast-fail
	int index;              // 用于记录遍历时外部类实例变量table(数组)中的下标
	Entry<K,V> current;     // current entry
	
	//构造方法中，从index=0开始遍历table数组，遇到null，index++，直到
	//遇到不为Null的Entry对象时，记录将当前Entry对象的引用保存到next中,
	//且index记录的是下一个下标的位置。
	HashIterator() {
		expectedModCount = modCount;
		if (size > 0) { // advance to first entry
			Entry[] t = table;
			while (index < t.length && (next = t[index++]) == null)
				;
		}
	}

	public final boolean hasNext() {
		return next != null;
	}
	
	//返回下一个Entry<K,V>对象
	final Entry<K,V> nextEntry() {
		if (modCount != expectedModCount)
			throw new ConcurrentModificationException();
		Entry<K,V> e = next;
		if (e == null)
			throw new NoSuchElementException();
		
		//如果Entry维护的链表的下一个结点是null，那么再次到table数组中遍历
		//找到不为null的Entry对象并返回
		//如果下一个结点不是null，就将下一个结点返回。
		if ((next = e.next) == null) {
			Entry[] t = table;
			while (index < t.length && (next = t[index++]) == null)
				;
		}
		//用current记录当前Entry对象。
		current = e;
		return e;
	}

	public void remove() {
		if (current == null)
			throw new IllegalStateException();
		if (modCount != expectedModCount)
			throw new ConcurrentModificationException();
		Object k = current.key;
		current = null;
		HashMap.this.removeEntryForKey(k);
		expectedModCount = modCount;
	}
}
```
紧接着的是该抽象类的三个实现：
4. `ValueIterator`
``` java
private final class ValueIterator extends HashIterator<V> {
	public V next() {
		return nextEntry().value;
	}
}
```
5. `KeyIterator`
``` java
private final class KeyIterator extends HashIterator<K> {
	public K next() {
		return nextEntry().getKey();
	}
}
```
6. `EntryIterator`
``` java
private final class EntryIterator extends HashIterator<Map.Entry<K,V>> {
	public Map.Entry<K,V> next() {
		return nextEntry();
	}
}
```
7. `KeySet extends AbstractSet<K>`:保存了key对象的Set。
8. `Values extends AbstractSet<V>`:保存了values对象的Set。
9. `EntrySet extends AbstractSet<Map.Entry<K,V>>`：保存了Entry对象的Set。

---
> 由于自己知识水平还不够，接下来本要写`JDK1.7 HashMap`·的方法的分析，但是有些方法自己都没理解明白，就更别说整理出来了。再加上`JDK1.8的HashMap`的Entry采用了`红黑树`来存储，我还是先学会红黑树再来吧。

推荐一个写的很不错的博客，分析了JDK1.7和1.8的HashMap的源码，而且有图，https://allenwu.itscoder.com/hashmap-analyse
