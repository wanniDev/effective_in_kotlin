# 아이템1. 가변성을 제한하라

```kotlin
class BankAccount {
	var balance = 0.0
	private set

	fun deposit(depositAmount: Double) {
		balance += depositAmount
	}
	
	@Throws(InsufficientFunds::class)
	fun withdraw(withdrawAmount: Double) {
		if (balance < withdrawAmount) {
			throw InsufficientFunds()
		}
		balance -= withdrawAmount
	}
}

fun main() {
  class InsufficientFunds : Exception()
  val account = BankAccount()
  println(account.balance) // 0.0
  
  account.deposit(100.0)
  println(account.balance) // 100.0
  
  account.deposit(50.0)
  println(account.balance) // 50.0
}
```

BankAccount에는 계좌에 잔액이 얼마인지를 나타내는 상태가 있다. 이 상태를 적절하게 관리하는 것은 생각보다 어렵다. 상태 관리를 어렵게하는 요소들은 다음과 같다.

1. 프로그램을 이해하고 디버그하기 힘들어진다.
2. 가변성이 있으면, 코드의 실행을 추론하기 어렵다.
3. 멀티스레드 환경에서는 적절한 동기화가 필요하다.
4. 테스트가 어렵다. 모든 상태를 테스트 해야 하므로, 변경이 많으면 많을 수록 더 많은 조합을 테스트 해야한다.
5. 상태 변경이 일어날 때 이러한 변경을 다른 부분에 달려야 하는 경우가 있다.

이렇게 가변성은 프로그래밍을 어렵게 하는 단점이 많아서 이를 완전하게 제한하는 프로그래밍 언어도 있다. 

가변성은 시스템의 상태를 나타내기 위한 중요한 방법이다. 하지만 변경이 일어나야 하는 부분을 신중하고 확실하게 결정하고 사용해야 한다.
## 코틀린에서 가변성 제한하기

코틀린은 가변성을 제한할 수 있게 설계되어 있다.

- 읽기 전용 프로퍼티(val)
- 가변 컬렉션과 읽기 전용 컬렉션 구분
- 데이터 클래스의 copy

### 읽기 전용 프로퍼티 (val)

- 마치 값(value)처럼 동작하며, 일반적인 방법으로는 값이 변하지 않는다. (읽고 쓸 수 있는 프로퍼티는 var로 만든다.)

```kotlin
val a = 10
a = 20 // compile error
```

- 읽기 전용 프로퍼티가 완전히 변경 불가능한 것은 아니다. mutalbe 객체를 담고 있다면, 내부적으로는 변경이 가능하다.

```kotlin
val list = mutableListOf(1,2,3)
list.add(4)
print(list) // [1, 2, 3, 4]
```

- 읽기 전용 프로퍼티는 다른 프로퍼티를 활용하는 사용자 정의 getter로도 정의할 수 있다. 정의 한 getter 내부에서 var 프로퍼티를 사용하는 val 프로퍼티는 var 프로퍼티가 변할 때마다 해당 값이 반영된다.

```kotlin
var name: String = "Kim"
var surname: String = "Dooli"
val fullName
	get() = "$name $surname"

fun main() {
  println(fullName) // Kim Dooli
	name = "GilDong"
  println(fullName) // Kim GilDong
}
```

- var는 getter와 setter를 모두 제공하지만, val은 변경이 불가능하므로 getter만 제공한다. 그러나, val은 오버라이드를 통해, var로 변경할 수 있다.

```kotlin
interface Element {
	val active: Boolean
}

class ActualElement: Element {
  override var active: Boolean = false
}
```

- val은 정의 옆에 상태가 바로 적히므로, 코드의 실행을 예측하는 것이 훨씬 간단하며, 스마트 캐스트 등의 추가적인 기능을 활용할 수 있다.
-  `fullName`은 getter로 정의했으므로 스마트 캐스트가 불가능하다. 값을 사용하는 시점의 name에 따라서 다른 결과가 나올 수 있기 때문이다.

```kotlin
var name: String? = "GilDong"
var surname: String = "Hong"
var fullName: String?
	get() = name?.let {"$it $surname"}

val fullName2: String? = name?.let {"$it $surname"}

fun main() {
	if (fullName != null) {
		println(funllName.length) // 오류
	}

	if (fullName2 != null) {
		println(fullName2.length) // 12(Kim GilDong)
	}
}
```

### 가변 컬렉션과 읽기 전용 컬렉션 구분하기

> 출처: [kotlinlang.com](http://kotlinlang.com/)

![image](https://user-images.githubusercontent.com/81374655/201474042-a7d40ba0-a42e-4fb5-a074-0bcafd70ec08.png)

- 인터페이스 읽기 전용: Iterable, Collection, Set, List
- 읽고 쓸 수 있는 컬렉션: MutableIterable, MutableCollection, MutableSet, MutableList
- 아래 예시 코드는 컬렉션 다운 캐스팅은 읽기 전용으로 리턴하였는데, 이는 읽기 전용의 제약을 위반하고 추상화를 무시하는 행위이다. 이런 코드는 안전하지 않고, 예측이 어렵다.

```kotlin
val list = listOf(1,2,3)

// NO!!!
if (list is MutableList) {
  list.add(4)
}
```

> 대신 이렇게 해보자

```kotlin
val list = listOf(1,2,3)
val mutableList = list.toMutableList()
mutableList.add(4)
```

### 데이터 클래스의 copy

- immutable 객체 사용의 장점
  1. 한 번 정의된 상태가 유지되므로, 코드를 이해하기 쉽다.
  2. immutable 객체는 공유했을 때도 충돌이 따로 이루어지지 않으므로, 병렬처리를 안전하게 할 수 있다.
  3. immutable 객체에 대한 참조는 변경되지 않으므로, 쉽게 캐시할 수 있다.
  4. immutable 객체는 방어적 복사본을 만들 필요가 없다.
  5. immutable 객체는 다른 객체를 만들 때 활용하기 좋다.
  6. immutable 객체는 'set' 또는 'map'의 키로 사용할 수 있다. 참고로 mutable 객체는 이러한 활용이 불가능하다. 해시 테이블은 처음 element를 넣을 때 element의 값을 기반으로 버킷을 결정하기 때문에, 변경이 일어나면 내부에서 요소를 찾을 수 없게 되어 버린다.

```kotlin
val names: SortedSet<FullName> = TreeSet()
val person = FullName("AAA", "AAA")
names.add(person)
names.add(FullName("BBB", "BBB"))
names.add(FullName("CCC", "CCC"))

print(names) // [AAA AAA, BBB BBB, CCC CCC]
print(person in names) // true

person.name = "ZZZ"
print(names) // [ZZZ AAA, BBB BBB, CCC CCC]
print(persin in names) // false
```

mutable 객체는 예측이 어렵고 위험하다는 단점이 있다. 반면 immutable 객체는 변경할 수 없다는 단점이 있다. 따라서 immutable 객체는 자신의 일부를 수정한 새로운 객체를 만들어내는 메서드를 가져야 한다.

```kotlin
class User (val name: String, val surname: String) {
  fun withSurname(surname: String) = User(name, surname)
}

var user = User("AAA", "BBB")
user = user.withSurname("CCC")
print(user) // User(name="AAA", surname="CCC")
```

- 모든 프로퍼티를 대상으로 이런 함수를 하나하나 만드는건 귀찮다. 이럴땐, data 한정자를 사용하면 된다.

```kotlin
data class User(val name:String, val surname: String)

var user = User("AAA", "BBB")
user = user.copy(surname = "CCC")
print(user) // User(name = "AAA", surname = "CCC")
```

## 다른 종류의 변경 가능 지점

변경할 수 있는 리스트를 만든다고 가정할 때

1. mutable 컬렉션 만들기
2. var로 읽고 쓸 수 있는 프로퍼티 만들기

```kotlin
val list1: MutableList<Int> = mutableListOf()
var list2: List<Int> = listOf()

lsit1.add(1)
list2 = list2 + 1

list1 += 1 // list1.plusAssign(1)로 변경
list2 += 1 // list2 = list2.plus(1)로 변경
```

- 1번 리스트의 경우 구체적인 리스트 구현 내부에 변경 가능 지점이 있어, 멀티스레드 처리가 이루어질 경우, 내부적으로 적절한 동기화가 되어있는지 알 수 없어 위험하다.
- 2번 리스트의 경우 프로퍼티 자체가 변경 가능 지점이라 멀티스레드 처리의 안정성이 좋다.

```kotlin
var list = listOf<Int>() 
	for (i in 1..1000) {
		thread {
      list = list + i
    }
	}
Thread.sleep(1000)
print(list.size) // 1~1000을 더한 값이 되지 않는다. 실행할 때마다 다른 숫자가 나온다
```

- mutable 리스트 대신 mutable 프로퍼티를 사용하는 형태는 사용자 정의 setter를 활용해서 변경을 추적할 수 있다.

```kotlin
var names by Delegates.observable(listOf<String>()) { _, old, new -> 
	println("Names changed from $old to $new")
}

names += "Fabio" // Names changed from [] to [Fabio]
names += "Bill" // Names changed from [Fabio] to [Fabio, Bill]
```

- 프로퍼티와 컬렉션을 모두 변경 가능한 지점으로 만드는 것은 최악의 방식이다.

```kotlin
// NO!!!
var list3 = mutableListOf<Int>()
```

### 변경 가능 지점 노출하지 말기

mutable 객체를 외부에 노출하는건 위험하다.

```kotlin
data class User(val name: String)

class UserRepository {
  private val storedUsers: MutableMap<Int, String> = mutableMapOf()
  
  fun loadAll() : MutableMap<Int, String> {
    return storedUsers
  }
}

val userRepository = UserRepository()

val sotredUsers = userRepository.loadAll()
sotredUsers[4] = "AAA"
// ...

print(userRepository.loadAll()) // {4=AAA}

```

처리하는 법

- 리턴하는 mutable 객체를 복제

```kotlin
class UserHolder {
  private val user: MutableUser()
  	return user.copy()
  //...
}
```

- 가변성을 제한 (컬렉션은 객체를 읽기 전용으로 슈퍼타입으로 업캐스트 하여 가변성을 제한한다.)

```kotlin
class UserRepository {
	private val storedUsers: MuatableMap<Int, String> = mutableMapOf()

	fun loadAll(): Map<Int,String> {
		return storedUsers
	}

	//...
}
```

## 정리

- var 보다는 val을 사용하는 것이 좋다.
- mutable 프로퍼티보다는 immutable 프로퍼티를 사용하는 것이 좋다.
- mutable 객체와 클래스보다는 immutable 객체와 클래스를 사용하는 것이 좋다.
- 변경이 필요한 대상을 만들어야 한다면, immutable 데이터 클래스로 만들고 copy를 활용하는 것이 좋다.
- 컬렉션에 상태를 저장해야 한다면, mutable 컬렉션 보다는 읽기 전용 컬렉션을 사용하는 것이 좋다.
- 변이 지점을 적절하게 설계하고, 불필요한 변이 지점은 만들지 않는 것이 좋다.
- mutable 객체를 외부에 노출하지 않는 것이 좋다.
- 가끔 효율성 이슈로 immutable 객체보다 mutable 객체를 사용하는 것이 좋을 때가 있다. (세부내용은 3부에서...)
- immutable 객체를 사용할 때 멀티스레드 때에 더 많은 주의를 기울어야 한다.