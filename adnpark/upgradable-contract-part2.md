# [업그레이더블 컨트랙트 씨-리즈] Part 2 - 프록시 컨트랙트 해체 분석하기

Created: June 1, 2022 12:47 AM

## **[업그레이더블 컨트랙트 씨-리즈 목차]**

Part 1 — 업그레이더블 컨트랙트란? 

***Part 2 — 프록시 컨트랙트(Proxy Contract) 해체 분석하기 (이번 글)***

Part 3 — 비콘 프록시 컨트랙트(Beacon Proxy Contract) 해체 분석하기

Part 4 — 미니멀 프록시 컨트랙트(Minimal Proxy Contract) 해체 분석하기

지난 글에서 업그레이더블 컨트랙트가 왜 등장했고, 어떻게 동작하는지 프록시 패턴을 통해 이해하는 시간을 가졌다. 글의 말미에서 업그레이드시에 실제로 변수가 저장되는 스토리지 영역에 문제가 없을지, 로직 컨트랙트를 계속 변경하면서 새로운 변수가 추가 되더라도 크게 문제가 없을지 등에 대한 의문을 남긴채로 마무리했었다. 

실제로 프록시 컨트랙트를 구현할 때는, 스토리지 충돌(Storage Collision)이라 부르는 위 이슈 뿐만 아니라 여러가지 주의해야 할 사항들이 있다. 따라서 이번 글에서는 프록시 컨트랙트를 구현할 때 주의해야 할 여러가지 사항들에 대해 알아보고, 이를 이해하기 위해 필요한 배경지식까지 친절하게 안내해드리고자 한다. 나아가 실무에서 자주 쓰이는 프록시 패턴들에 대해 살펴보고자 한다. 또한 마지막으로 간단한 컨트랙트를 프록시 패턴을 활용하여 업그레이더블 컨트랙트로 구현해보는 실습을 함께 해보자.

먼저 프록시 컨트랙트를 구현할 때 주의해야 할 사항들부터 살펴보도록 하자

첫번째 이슈는 스토리지 충돌이다. 프록시 컨트랙트에서 스토리지 충돌이 왜 발생하는지 이해하기 위해서는 이더리움 어카운트의 스토리지에 데이터들이 저장되는 방식에 대한 이해가 필요하다. 

### 스토리지 레이아웃(Storage Layout)

<aside>
💡 스토리지 슬롯에 관한 모든것들을 이해하면 당연히 좋겠지만, 이 글의 목적은 업그레이더블 컨트랙트에서 스토리지 충돌이 왜 발생하는지, 그리고 그것을 어떻게 피해갈 수 있는지이다. 이 목적을 염두에 두고 글을 읽어보자.
더 자세한 내용은 [솔리디티 심화 씨-리즈] 스토리지 레이아웃과 가스 최적화 글에서 다룰 예정이다.

</aside>

우리가 솔리디티로 컨트랙트를 작성할 때, 여러가지 상태 변수를 선언하고 사용하게 된다. 이 때 상태 변수들이 저장되는 방식을 솔리디티에서는 스토리지 레이아웃이라고 부른다.

스토리지 레이아웃의 가장 기본이 되는 단위는 슬롯으로, 모든 데이터들이 이 슬롯을 기준으로 관리된다.

스토리지 슬롯이 관리되는 규칙은 다음과 같다. 

- 슬롯은 32바이트 크기를 갖는다.
- 일반적으로 변수가 선언된 순서대로 슬롯에 할당된다.
- 해당 변수 타입의 크기만큼 슬롯을 사용하며, 다음 변수 타입이 기존 슬롯에서 다 포함할 수 없는 경우 다음 슬롯에 할당된다.
    - 단, 동적 배열(Dynamically-sized array)과 매핑(Mapping) 타입은 예외이다.
    - 동적 배열의 경우 배열이 시작해야 하는 슬롯에는 오직 해당 배열의 길이(Length)만 저장한다. 실제 배열 원소들은 배열이 위치한 슬롯 넘버를 keccak256으로 해싱한 값이 된다.
    - 매핑의 값(value)은 매핑이 시작되는 슬롯 넘버와 key값을 동일하게 해싱한 값의 슬롯 넘버에 위치하게 된다.

스토리지 슬롯 모델은 *본질적으로 32 또는 64 비트 크기를 기준으로하는 Virtual RAM 모델과 크게 다르지 않다. 다만 최소 크기 단위가 256 비트라는 점만 다르다.*

규칙이 뭔가 많고 복잡해 보이는데, 실제 코드 예시를 보면서 이해하면 생각보다 쉽다. 

```solidity
uint256 foo;
uint256 bar;
uint256[] items;
mapping(uint256 => uint256) values;

function allocate() public {
	require(0 == items.length);
	// allocate array items
	items.length = 2;
	items[0] = 12;
	items[1] = 42;
	// allocate mapping values
	values[0] = 100;
	values[1] = 200;
}
```

위와 같은 컨트랙트가 있고, allocate() 함수를 호출하여 상태 변수에 값을 저장한다고 할 때, 해당 컨트랙트의 스토리지는 다음과 같다.

![Untitled](%5B%E1%84%8B%E1%85%A5%E1%86%B8%E1%84%80%E1%85%B3%E1%84%85%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%83%E1%85%A5%E1%84%87%E1%85%B3%E1%86%AF%20%E1%84%8F%E1%85%A5%E1%86%AB%E1%84%90%E1%85%B3%E1%84%85%E1%85%A2%E1%86%A8%E1%84%90%E1%85%B3%20%E1%84%8A%E1%85%B5-%E1%84%85%E1%85%B5%E1%84%8C%E1%85%B3%5D%20Part%202%20-%20%E1%84%91%E1%85%B3%E1%84%85%E1%85%A9%E1%86%A8%206bd9b986b2f8479e9237167c4f92b8f9/Untitled.png)

256 비트(32 바이트)크기로 변수가 선언된 순서대로 슬롯에 차곡차곡 할당되는 모습을 확인할 수 있다. 동적 배열이 저장되는 방식, 그리고 매핑의 값들이 매핑이 선언된 순서의 슬롯 넘버와 키값을 해싱한 값의 슬롯 넘버에 저장되는것에 주목해보자. **특히 여기서 매핑 값들이 저장되는 방식이 이후 프록시 패턴에서의 스토리지 충돌을 해결하기 위한 중요한 단초가 된다.**

*여기서 매핑의 데이터가 산발적으로 저장되다보면 데이터간의 충돌이 발생하지 않을까하고 걱정할 수 있지만, 이는 256 비트 해시 함수의 충돌 가능성이 무시 가능한 수준이기 때문에, 매우 높은 확률로 문제가 없다고 할 수 있다.*

예시 출처: [https://mixbytes.io/blog/collisions-solidity-storage-layouts](https://mixbytes.io/blog/collisions-solidity-storage-layouts)

- 상속된 변수들은 어떻게 되는건지 체크 필요

For contracts that use inheritance, the ordering of state variables is determined by the C3-linearized order of contracts starting with the most base-ward contract. If allowed by the above rules, state variables from different contracts do share the same storage slot.

- 스토리지 충돌(Storage Collision)
    - Storage Slot 구조
    - EIP-1967
- Initializing code
    - Initializer
    - init code 구조
- Function clashes
    - msg.sender check

## 스토리지 충돌(Storage Collision)

## 생성자 초기화 코드(Initializing Constructor Code)

## 함수 충돌(Function Clashes)

## Transparent vs UUPS

흔히 사용되는 프록시 패턴에는 Transparent 패턴과 UUPS(Universal Upgradeable Proxy Standard) 패턴이 있다. 두가지 패턴 모드 업그레이더블 컨트랙트를 위한 프록시 패턴이라는 점에서 공통점이 있다. 그렇다면 차이점은 어디에 있을까? 바로 업그레이드 로직이 위치하는 곳에 있다. 업그레이드 로직이 프록시 컨트랙트에 위치하느냐, 로직 컨트랙트에 위치하느냐에 있다. 

Transparent 프록시 패턴은 업그레이드 관련 로직이 프록시 컨트랙트에 위치한다. 이는 곧 프록시 컨트랙트를 배포하는 비용이 조금 더 비싸지는 것을 의미하며, 

예를 들어 동일한 로직 컨트랙트를 사용하는 100개의 프록시 컨트랙트를 배포한다고 해보자. 이 때 업그레이드 관련 코드가 100번 반복해서 추가 된다고 하면, 그만큼 피같은 돈인 가스가 소모된다는 것을 의미한다. 또한, 통상적으로 업그레이드 기능은 필연적으로 스마트 컨트랙트의 탈중앙성을 훼손하게 된다. 관리자에 의한 업그레이드 기능이 사용자의 자산을 위협하는 형태로 활용될 여지가 있기 때문이다. Transparent 패턴은 프록시 컨트랙트가 업그레이드 로직을 포함하기 때문에, 영원히 이 기능은 프록시 컨트랙트에서 제거될 수 없다. 따라서 영구적으로 컨트랙트의 탈중앙성을 훼손하게 된다.

*흥미로운 사실: 업그레이드 기능 없이, 오직 동일한 로직 컨트랙트를 여러번 활용하기 위한 목적으로 사용되는 프록시 패턴을 미니멀 프록시 컨트랙트라고 한다. 이에 대한 자세한 내용은 Part 4 글에서 다룰 예정이다.*

```solidity
// TODO: add contract example
```

UUPS는 업그레이드 로직이 구현체, 즉 로직 컨트랙트에 위치하게 된다. 

오픈제플린에서도 Transparent가 아닌 UUPS 패턴 사용을 권장하고 있다.

*The original proxies included in OpenZeppelin followed the [Transparent Proxy Pattern](https://blog.openzeppelin.com/the-transparent-proxy-pattern/)
. While this pattern is still provided, our recommendation is now shifting towards UUPS proxies, which are both lightweight and versatile.*

더 읽어보기

- ****Proxy Patterns For Upgradeability Of Solidity Contracts: Transparent vs UUPS Proxies****

[https://mirror.xyz/0xB38709B8198d147cc9Ff9C133838a044d78B064B/M7oTptQkBGXxox-tk9VJjL66E1V8BUF0GF79MMK4YG0](https://mirror.xyz/0xB38709B8198d147cc9Ff9C133838a044d78B064B/M7oTptQkBGXxox-tk9VJjL66E1V8BUF0GF79MMK4YG0)

[https://mixbytes.io/blog/collisions-solidity-storage-layouts](https://mixbytes.io/blog/collisions-solidity-storage-layouts)

[https://blog.logrocket.com/creating-contract-factory-clone-solidity-smart-contracts/](https://blog.logrocket.com/creating-contract-factory-clone-solidity-smart-contracts/)