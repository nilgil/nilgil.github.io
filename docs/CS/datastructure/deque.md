---
title: Deque
parent: Data Structure
---

# Deque
양쪽 끝에서 삽입과 삭제가 모두 가능하며 Stack과 Queue의 특성을 모두 가진다.



## Java Deque API

### 1. Deque 주요 구현체
- **ArrayDeque\<E>**: 일반적인 용도로 가장 많이 사용
- **LinkedList\<E>**: 메모리 오버헤드가 있지만 중간 삽입/삭제가 필요할 때 사용
- **ConcurrentLinkedDeque\<E>**: Thread-safe가 필요할 때 사용

### 2. 주요 API와 시간 복잡도

- First Element (Head) 작업 메서드

| 메서드 | 시간 복잡도 | 설명 |
|--------|------------|------|
| `void addFirst(E e)` | O(1) | 맨 앞에 요소 추가, 공간 부족 시 IllegalStateException |
| `boolean offerFirst(E e)` | O(1) | 맨 앞에 요소 추가, 성공 시 true 반환 |
| `E removeFirst()` | O(1) | 맨 앞 요소 제거 후 반환, 비어있으면 NoSuchElementException |
| `E pollFirst()` | O(1) | 맨 앞 요소 제거 후 반환, 비어있으면 null |
| `E getFirst()` | O(1) | 맨 앞 요소 반환, 제거 안함, 비어있으면 NoSuchElementException |
| `E peekFirst()` | O(1) | 맨 앞 요소 반환, 제거 안함, 비어있으면 null |

- Last Element (Tail) 작업 메서드

| 메서드 | 시간 복잡도 | 설명 |
|--------|------------|------|
| `void addLast(E e)` | O(1) | 맨 뒤에 요소 추가, 공간 부족 시 IllegalStateException |
| `boolean offerLast(E e)` | O(1) | 맨 뒤에 요소 추가, 성공 시 true 반환 |
| `E removeLast()` | O(1) | 맨 뒤 요소 제거 후 반환, 비어있으면 NoSuchElementException |
| `E pollLast()` | O(1) | 맨 뒤 요소 제거 후 반환, 비어있으면 null |
| `E getLast()` | O(1) | 맨 뒤 요소 반환, 제거 안함, 비어있으면 NoSuchElementException |
| `E peekLast()` | O(1) | 맨 뒤 요소 반환, 제거 안함, 비어있으면 null |

- Queue/Stack 작업 메서드

| 메서드 | 시간 복잡도 | 설명 |
|--------|------------|------|
| `boolean add(E e)` | O(1) | addLast()와 동일 |
| `boolean offer(E e)` | O(1) | offerLast()와 동일 |
| `E remove()` | O(1) | removeFirst()와 동일 |
| `E poll()` | O(1) | pollFirst()와 동일 |
| `E element()` | O(1) | getFirst()와 동일 |
| `E peek()` | O(1) | peekFirst()와 동일 |
| `void push(E e)` | O(1) | addFirst()와 동일 |
| `E pop()` | O(1) | removeFirst()와 동일 |

- 검색 및 기타 작업 메서드

| 메서드 | 시간 복잡도 | 설명 |
|--------|------------|------|
| `boolean contains(Object o)` | O(n) | 요소 검색 |
| `int size()` | O(1) | 크기 반환 |
| `void clear()` | O(n) | 모든 요소 제거 |
| `Object[] toArray()` | O(n) | 배열로 변환 |
| `Iterator<E> iterator()` | O(1) | 순방향 반복자 반환 |
| `Iterator<E> descendingIterator()` | O(1) | 역방향 반복자 반환 |

### 3. 일반적인 활용 예시

- 양방향 버퍼로 사용

```java
Deque<String> buffer = new ArrayDeque<>();
buffer.addFirst("우선처리");
buffer.addLast("나중처리");
```

- 스택으로 사용

```java
Deque<String> stack = new ArrayDeque<>();
stack.push("First");
stack.push("Second");
String top = stack.pop();
```

- 큐로 사용

```java
Deque<String> queue = new ArrayDeque<>();
queue.offer("First");
queue.offer("Second");
String first = queue.poll();
```

- 슬라이딩 윈도우 구현

```java
Deque<Integer> slidingWindow = new ArrayDeque<>();
for (int i = 0; i < 5; i++) {
    slidingWindow.addLast(i);
    if (slidingWindow.size() > 3) {
        slidingWindow.removeFirst();
    }
}
```
