```java
    arrays.stream()
    .collect(Collectors.groupBy(a->a.getField,Collectors.counting())
    .entrySet.stream()
    .filter(entry->entry.getValue()>1)
    .map(entry->entry.getKey())
    .collect(Collectors.toList());
```