---
layout: post
title: List及Set集合
date: 2019-10-28
tags: [Collection集合]
---

**Collection包含List和Set**

**List集合**
1. List集合中的元素有序且可重复
2. ArrayList底层结构为数组，非线程安全，查询效率高，增删效率低
3. LinkedList底层结构为链表，非线程安全，增删效率高，查询效率较低
4. Vector底层结构为数组，线程安全，查询快，增删慢

**Set集合**
1. Set集合中的元素唯一不可重复，通过hashCode()和equals()来保证元素的唯一性
2. HashSet底层结构为哈希表
3. LinkedHashSet底层结构为链表加哈希表，元素的插入是有序的，由链表来保证元素有序，哈希表保证元素的唯一性
4. TreeSet底层结构是红黑树，能够对元素进行排序（自然排序和选择排序），元素不能为null;

```java
public static void main(String[] args) {

        //唯一无序
        HashSet<String> hashSet = new HashSet<>();
        
        //唯一插入顺序排序
        LinkedHashSet<String> linkedHashSet = new LinkedHashSet<>();
        
        //唯一内部实现排序
        TreeSet<String> treeSet = new TreeSet<>();

        for(String s : Arrays.asList("E", "A", "B")) {
            hashSet.add(s);
            linkedHashSet.add(s);
            treeSet.add(s);
        }
        System.out.println("hashSet : " + hashSet);
        System.out.println("linkedHashSet : " + linkedHashSet);
        System.out.println("treeSet : " + treeSet);

    }
```
输出结果：
```xml
hashSet : [A, B, E]
linkedHashSet : [E, A, B]
treeSet : [A, B, E]
```

**TreeSet的内部排序实现方式**

1. 当TreeSet内的元素类型为基本数据类型时，默认按照元素的Ascll值进行排序
2. 当TreeSet内的元素为引用类型时（自定义对象），采用自然排序和选择排序两种方式
3. 自然排序：自定义对象需要实现Compareable接口,重写compareTo方法

```java
public class DemoTest {

    public static void main(String[] args) {

        Student s1 = new Student(21, "cc");
        Student s2 = new Student(20, "dd");
        Student s3 = new Student(22, "aa");
        TreeSet<Student> treeSet = new TreeSet<>();
        treeSet.add(s1);
        treeSet.add(s2);
        treeSet.add(s3);
        treeSet.forEach(s -> System.out.println(s.toString()));

    }
}

//实现Comparable接口，重写compareTo方法
class Student implements Comparable<Student> {

    private Integer age;

    private String name;

    public Student() {
    }

    public Student(Integer age, String name) {
        this.age = age;
        this.name = name;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "Student{" +
                "age=" + age +
                ", name='" + name + '\'' +
                '}';
    }

    @Override
    public int compareTo(Student s) {

        //return -1; //-1表示放在红黑树的左边,即逆序输出
        //return 1;  //1表示放在红黑树的右边，即顺序输出
        //return o;  //表示元素相同，仅存放第一个元素
        return this.age - s.getAge();
    }
}

```
排序后输出：
```xml
Student{age=20, name='dd'}
Student{age=21, name='cc'}
Student{age=22, name='aa'}
```

4. 选择排序:自定义排序比较器（实现Comparator接口，重写conpare方法）,在初始化TreeSet时传入自定义的比较器

```java
public class DemoTest {

    public static void main(String[] args) {

        Student s1 = new Student(21, "cc");
        Student s2 = new Student(20, "dd");
        Student s3 = new Student(22, "aa");
        
        //传入自定义的比较器
        TreeSet<Student> treeSet = new TreeSet<>(new Comparator<Student>() {
                    @Override
                    public int compare(Student o1, Student o2) {
                        return o1.getAge() - o2.getAge();
                    }
                });
        treeSet.add(s1);
        treeSet.add(s2);
        treeSet.add(s3);
        treeSet.forEach(s -> System.out.println(s.toString()));

    }
}

class Student {

    private Integer age;

    private String name;

    public Student() {
    }

    public Student(Integer age, String name) {
        this.age = age;
        this.name = name;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "Student{" +
                "age=" + age +
                ", name='" + name + '\'' +
                '}';
    }

}

```

