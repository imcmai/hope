## LAMBDA取出对象list中某个字段重复的集合
```java
    arrays.stream()
    .collect(Collectors.groupBy(a->a.getField,Collectors.counting())
    .entrySet.stream()
    .filter(entry->entry.getValue()>1)
    .map(entry->entry.getKey())
    .collect(Collectors.toList());
```