# [업그레이더블 컨트랙트 씨-리즈] Part 2 - 프록시 컨트랙트(Proxy Contract) 해체 분석하기

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

### 스토리지 슬롯(Storage Slot)

우리가 솔리디티로 컨트랙트를 작성할 때, 여러가지 상태 변수를 선언하고 사용하게 된다. 이 때 변수들이 저장되는 방식으로 솔리디티는 슬롯이라는 개념을 사용하고 있다. 

스토리지 슬롯이 관리되는 규칙은 다음과 같다.

- 슬롯은 32바이트 크기를 갖는다.
- 일반적으로 변수가 선언된 순서대로 슬롯의 사이즈에 맞게 할당된다.
    - 단, 동적 배열과 매핑 타입은 예외이다.
- 

슬롯은 32바이트 크기로, 변수가 선언된 순서대로 슬롯의 사이즈에 맞게 저장되는 방식이다. 실제 코드를 보면서 이해해보자.

```solidity

```

- The first item in a storage slot is stored lower-order aligned.
- Value types use only as many bytes as are necessary to store them.
- If a value type does not fit the remaining part of a storage slot, it is stored in the next storage slot.
- Structs and array data always start a new slot and their items are packed tightly according to these rules.
- Items following struct or array data always start a new storage slot.

For contracts that use inheritance, the ordering of state variables is determined by the C3-linearized order of contracts starting with the most base-ward contract. If allowed by the above rules, state variables from different contracts do share the same storage slot.

- 상속된 변수들은 어떻게 되는건지 체크 필요

### Transparent vs UUPS

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

- 스토리지 충돌(Storage Collision)
    - Storage Slot 구조
    - EIP-1967
- Initializing code
    - Initializer
    - init code 구조
- Function clashes
    - msg.sender check

이를 이해하기 위해서는 이더리움 어카운트의 스토리지가 데이터를 저장하는 방식을 먼저 이해하고 있어야 한다.

- 지난 글 이어읽기
- 스토리지 충돌, 어카운트 스토리지 구조

더 읽어보기

- ****Proxy Patterns For Upgradeability Of Solidity Contracts: Transparent vs UUPS Proxies****

[https://mirror.xyz/0xB38709B8198d147cc9Ff9C133838a044d78B064B/M7oTptQkBGXxox-tk9VJjL66E1V8BUF0GF79MMK4YG0](https://mirror.xyz/0xB38709B8198d147cc9Ff9C133838a044d78B064B/M7oTptQkBGXxox-tk9VJjL66E1V8BUF0GF79MMK4YG0)

[https://mixbytes.io/blog/collisions-solidity-storage-layouts](https://mixbytes.io/blog/collisions-solidity-storage-layouts)