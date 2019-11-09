### Java 8 Stream API

stream即是流，是可变的、无固定形态的。

Collection接口下的List和Set都可以看做是固定结构的数据，也就是形态固定、不可变（结构），Java 8提供了将固定结构的数据生成流的特性，这样一来，就可以对这些数据进行许多聚合操作，以此改变数据的内容或形状。

Java中，`java.util.stream.Stream`类定义了支持的流操作，大部分操作可以分为**中间操作（intermediate）**或者**终结操作（terminal）**。中间操作返回的仍是stream，终结操作返回的是确定类型的结果。

#### stream特点

- 不是一种数据结构
- 用来支持lambda
- 不支持下标访问
- 可以输出为array或者list
- 并行操作



#### 创建stream

- Stream.of(val1, val2, val3...)

  ```java
  public static void main(String[] args) {
      Stream<Integer> st = Stream.of(1, 2, 3, 4, 5, 6, 7);
      st.forEach(System.out::println);
  }
  ```

- Stream.of(arrayOfElement)

  ```java
  public static void main(String[] args) {
      Stream<Integer> st = Stream.of(new Integer[] {1,2,3,4,5});
      st.forEach(System.out::println);
  }
  ```

- List.stream()

  ```java
  public static void main(String[] args) {
      List<Integer> list = new ArrayList<>();
      for (int i = 0; i < 10; i++) {
          list.add(i + 1);
      }
      list.stream().forEach(System.out::println);
  }
  ```

- Set.stream()



#### 从stream中生成collection

`java.util.stream.collectors`的javadoc给出了样例

// Accumulate names into a List
     List<String> list = people.stream().map(Person::getName).collect(Collectors.toList());

```java
 // Accumulate names into a TreeSet
 Set<String> set = people.stream().map(Person::getName).collect(Collectors.toCollection(TreeSet::new));

 // Convert elements to strings and concatenate them, separated by commas
 String joined = things.stream()
                       .map(Object::toString)
                       .collect(Collectors.joining(", "));

 // Compute sum of salaries of employee
 int total = employees.stream()
                      .collect(Collectors.summingInt(Employee::getSalary)));

 // Group employees by department
 Map<Department, List<Employee>> byDept
     = employees.stream()
                .collect(Collectors.groupingBy(Employee::getDepartment));

 // Compute sum of salaries by department
 Map<Department, Integer> totalByDept
     = employees.stream()
                .collect(Collectors.groupingBy(Employee::getDepartment,
                                               Collectors.summingInt(Employee::getSalary)));

 // Partition students into passing and failing
 Map<Boolean, List<Student>> passingFailing =
     students.stream()
             .collect(Collectors.partitioningBy(s -> s.getGrade() >= PASS_THRESHOLD));
```


#### stream核心操作

##### 中间操作

- Stream.filter()

  对流中的每个元素执行筛选，用到的Predicate类型的函数，也就是filter内的函数必须返回boolean

  ```java
  list.stream()
      .filter(i -> i % 2 == 0)
      .forEach(System.out::println);
  // 筛选出偶数
  ```

  ```java
  list.stream()
      .filter(s -> s.startsWith("A"))
      .forEach(System.out::println);
  // 筛选出首字符为A的元素
  ```

- Stream.map()

  对流中的元素进行操作，产生新的中间态

  ```java
  list.stream()
  	.filter(s -> s.startsWith("A"))
  	.map(str -> str.toUpperCase())
  	.forEach(System.out::println);
  // 筛选出首字符为A的元素，并将这些元素都变为大写
  ```

  ```java
  list.stream()
      .filter(s -> s.startsWith("A"))
      .map(str -> str.length())
      .forEach(System.out::println);
  // 改变中间元素的类型，筛选出首字符为A的元素，将其长度作为新的中间元素打印输出
  ```

- Stream.sorted()

  ```java
  list.stream()
      .sorted()
      .forEach(System.out::println);
  // 默认是自然排序
  ```

  ```java
  list.stream()
      .sorted((o1, o2) -> o1.length() - o2.length())
      .forEach(System.out::println);
  // 可以传入Comparator接口自定义排序规则
  ```

##### 终结操作

- Stream.forEach()

  遍历所有元素，对其执行一些操作

  ```java
  list.forEach(System.out::println);
  // 打印
  ```

  ```java
  map.entrySet().forEach(entry -> {
      System.out.println("key: " + entry.getKey());
      System.out.println("Value: " + entry.getValue());
  });
  // 打印Map
  ```

- Stream.collect()

  [使用方法看这里](#从stream中生成collection)

- Stream.match()

  对流中的元素进行选择匹配，返回boolean表示匹配是否成功

  ```java
  boolean anyMatch = list.stream()
      .anyMatch(s -> s.startsWith("A"));
  ```

  ```java
  boolean allMatch = list.stream()
      .allMatch(s -> s.startsWith("A"));
  ```

  ```java
  boolean noneMatch = memberNames.stream()
      .noneMatch(s -> s.startsWith("A"));
  ```

- Stream.count()

  ```java
  long count = memberNames.stream()
      .filter(s -> s.startsWith("A"))
      .count();
  ```

- Stream.reduce()

  对流中的元素进行两两操作，最终合并成一个Optional类型的元素

  ```java
  Optional<String> reduced = list.stream()
  	.reduce((s1,s2) -> s1 + ``"#"` `+ s2);
  reduced.ifPresent(System.out::println);
  // 结果是将list中的所有元素用#拼接
  ```

##### 其他操作

​	Stream.findFirst()

​	一般用在filter过滤之后返回第一个元素

```java
String firstMatchedName = memberNames.stream()
    .filter((s) -> s.startsWith(``"L"``))
    .findFirst().get();
```



#### 并行流

Java 7中假如了Fork/Join特性之后，可以轻松的实现并行操作，但直接使用Fork/Join比较麻烦，还需要对其原理有一定了解。

Stream中提供了便捷的并行流方法，用以提高计算效率

```java
public static void main(String[] args) {
    List<String> list = new ArrayList<>();
    String str = "yoha";
    for (int i=0; i<100000; i++) {
        list.add(str+i);
    }
    normalCompute(list);
    parallelCompute(list);
}

private static void parallelCompute(List<String> list) {
    long st = System.currentTimeMillis();
    long count = list.parallelStream()
        .map(str -> str + "test")
        .map(str -> str.toUpperCase())
        .count();
    long ed = System.currentTimeMillis();
    System.out.println("parallelCompute: " + (ed - st));
}

private static void normalCompute(List<String> list) {
    long st = System.currentTimeMillis();
    long count = list.stream()
        .map(str -> str + "test")
        .map(str -> str.toUpperCase())
        .count();
    long ed = System.currentTimeMillis();
    System.out.println("normalCompute: " + (ed - st));
}
```

```java
output: normalCompute: 123
	    parallelCompute: 15
```

可以看到效率显著提升。

