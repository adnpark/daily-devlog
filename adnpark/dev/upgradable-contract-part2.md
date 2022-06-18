# [업그레이더블 컨트랙트 씨-리즈] Part 2 - 프록시 컨트랙트 해체 분석하기

Created: 2022년 6월 1일 오전 12:47

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

*여기서 매핑의 데이터가 산발적으로 저장되다보면 데이터간의 충돌이 발생하지 않을까하고 걱정할 수 있지만, 이는 256 비트 해시 함수의 해시 충돌 가능성이 무시 가능한 수준이기 때문에, 매우 높은 확률로 문제가 없다고 할 수 있다.*

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

솔리디티의 스토리지 레이아웃, 특히 스토리지 슬롯의 구조에 대해 이해했으니 이제 스토리지 충돌이 무엇이고, 왜 발생하는지 알아볼 차례다. 

먼저 아래와 같은 프록시 컨트랙트와 로직 컨트랙트가 있다고 해보자.

```solidity
contract Proxy {
	address implementation
	// ...
}

contract Implementation {
	address foo;
	uint256 bar;
	// ...
}
```

각 컨트랙트는 아래와 같은 스토리지 레이아웃으로 구성될 것이다.

![Untitled](%5B%E1%84%8B%E1%85%A5%E1%86%B8%E1%84%80%E1%85%B3%E1%84%85%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%83%E1%85%A5%E1%84%87%E1%85%B3%E1%86%AF%20%E1%84%8F%E1%85%A5%E1%86%AB%E1%84%90%E1%85%B3%E1%84%85%E1%85%A2%E1%86%A8%E1%84%90%E1%85%B3%20%E1%84%8A%E1%85%B5-%E1%84%85%E1%85%B5%E1%84%8C%E1%85%B3%5D%20Part%202%20-%20%E1%84%91%E1%85%B3%E1%84%85%E1%85%A9%E1%86%A8%206bd9b986b2f8479e9237167c4f92b8f9/Untitled%201.png)

프록시 컨트랙트는 `delegatecall`을 활용하여 로직 컨트랙트의 함수를 호출한다. 이는 곧 로직 컨트랙트의 함수를 프록시 컨트랙트의 컨텍스트에서 실행한다는것을 의미하므로, 로직 컨트랙트에서 `foo` 변수를 수정하게 되면 이는 곧 `implementation` 변수를 수정하게 된다는것을 의미한다. `implementation` 변수는 로직 컨트랙트의 주소를 저장하는 아주 중요한 부분이다. 하지만, 이 변수에 예상치 못한 다른 값이 할당되게 되면 해당 프록시 컨트랙트는 더 이상 제 기능을 하지 못하게 될 것이다. 이처럼 프록시 컨트랙트에서의 의도치 않은 스토리지 충돌은 컨트랙트에 치명적 결함을 야기할 수 있다.

그렇다면 이는 어떻게 해결할 수 있을까? 프록시 컨트랙트는 어떠한 스토리지 슬롯도 사용해서는 안되는것일까? 이를 해결하기 위한 방법이 바로 [EIP-1967](https://eips.ethereum.org/EIPS/eip-1967)인 Standard Proxy Storage Slots, 즉 표준 프록시 스토리지 슬롯이다.EIP-1967의 아이디어는 매우 단순하다.

스토리지 슬롯을 순차적으로 사용하면 충돌 가능성이 높으니, 충돌 가능성이 무시 가능한 수준에서 랜덤에 가깝게(pseudo-random) 배정하면 된다는 것이다.

*위 스토리지 레이아웃 파트에서 매핑이 스토리지 슬롯에 저장되는 방식이 어떠했는지 기억하는가? 바로 매핑이 선언된 슬롯의 인덱스 넘버와 매핑의 키값을 해싱하여 밸류가 저장되는 슬롯 위치를 결정하였다. 이와 비슷한 아이디어를 적용해보자는 것이다!*

구체적인 아이디어는 이러하다. 저장하고 싶은 변수의 이름을 keccak256으로 해싱한 후 1을 뺀 값을 슬롯 넘버로 사용하는 것이다. 이 때 주의해야 할 점은 해싱에 사용되는 값을 **절대로 중복해서 사용하지 않아야 한다**는 것이다. 그렇지 않다면 동일하게 스토리지 충돌이 발생하게 된다.

```solidity
bytes32 internal constant _IMPLEMENTATION_SLOT = bytes32(uint256(
  keccak256('eip1967.proxy.implementation')) - 1
));
```

그런데 슬롯 넘버는 어떻게 지정하고, 또 해당 슬롯에 있는 값은 어떻게 읽고 쓰는것일까? 이는 오픈제플린의 [ERC1967 관련 컨트랙트](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/proxy/ERC1967/ERC1967Upgrade.sol)와 [StorageSlot 컨트랙트](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/StorageSlot.sol)에서 다루고 있다.

```solidity
abstract contract ERC1967Upgrade {
	bytes32 internal constant _IMPLEMENTATION_SLOT = bytes32(uint256(
	  keccak256('eip1967.proxy.implementation')) - 1
	));

	function _getImplementation() internal view returns (address) {
		return StorageSlot.getAddressSlot(_IMPLEMENTATION_SLOT).value;
	}

	function _setImplementation(address newImplementation) private {
		require(Address.isContract(newImplementation), "ERC1967: new implementation is not a contract");
		StorageSlot.getAddressSlot(_IMPLEMENTATION_SLOT).value = newImplementation;
	}
}

library StorageSlot {
	function getAddressSlot(bytes32 slot) internal pure returns (AddressSlot storage r) {
		/// @solidity memory-safe-assembly
		assembly {
			r.slot := slot
		}
	}
}
```

흔히 볼 수 있는 Getter & Setter 패턴과 크게 다르지 않다. 차이점이라면 솔리디티 어셈블리를 통해 해당 슬롯에 위치한 변수를 로우 레벨에서 읽고 쓴다는 것이다. 

이처럼 EIP-1967을 이용하면 프록시 컨트랙트에서 변수를 저장하면서, 동시에 안전하게 로직 컨트랙트를 사용하고 업그레이드도 무리 없이 진행할 수 있다.

### 로직 컨트랙트간의 스토리지 충돌

하지만 EIP-1967으로도 피해갈 수 없는 스토리지 충돌이 존재한다. 바로 이전 버전의 로직 컨트랙트와 업그레이드된 새로운 버전의 로직 컨트랙트의 스토리지 충돌이다. 코드 예시로 빠르게 살펴보자.

```solidity
contract V1 {
	address foo;
	uint256 bar;
	// ...
}

contract V2 {
	address baz;
	address foo;
	uint256 bar;
}
```

로직 컨트랙트를 V1에서 V2로 업그레이드 한다고 가정하자. V2에서는 새롭게 address baz라는 상태 변수가 추가 되었다. 그런데, baz의 선언 위치를 기존의 변수보다 앞에 두었다. 이렇게 하면 어떤 일이 생길까? 스토리지 레이아웃을 충분히 이해했다면, 기존 address foo의 슬롯에 address baz의 슬롯이 할당된다는 것을 알 수 있다. 즉, 스토리지 충돌이 발생하게 되는 것이다. 물론 다른 슬롯의 순서에도 변경이 있었기 때문에, 단순히 baz 변수 하나에만 영향을 미치는것은 아니다. 

이러한 충돌을 피하기 위해서는 업그레이드시 상태 변수의 선언 순서에 주의를 기울여야 한다.

```solidity
contract V1 {
	address foo;
	uint256 bar;
	// ...
}

contract V2 {
	address foo;
	uint256 bar;
	address baz;
}
```

위와 같이 기존 변수의 뒤에 새로운 상태 변수를 선언하면 스토리지 충돌은 발생하지 않는다. 이처럼 상태 변수들의 선언 위치를 하나하나 신경 쓰는것은 상당한 주의를 요한다. 때문에 실제로 업그레이더블 컨트랙트를 작성할 때는 기존의 로직 컨트랙트를 상속해서 작성하는것이 일반적이다.

TODO: 상속하는것이 일반적인 이유 추가하기

## 생성자 초기화 코드(Initializing Constructor Code)

프록시 패턴에서는 생성자(Constructor)를 사용할 수 없다. 프록시 패턴은 스토리지를 담당하는 프록시 컨트랙트와 실제 구현을 담당하는 로직 컨트랙트로 나뉜다. 생성자 함수는 컨트랙트 배포시에만 단 한번 호출되고 런타임 바이트코드에 포함되지 않으므로, 프록시 컨트랙트는 이를 호출할 수 없다. 

그렇다면 프록시 패턴에서는 생성자와 같이 컨트랙트 배포시에 실행되는 코드를 활용할 수 없는것일까? 다행히도 이 역시 피해갈 수 있는 방법이 있다. 소위 약간 짜치는 방식인데, 생성자 코드를 initializer라고 하는 함수로 옮기고, 해당 함수가 컨트랙트 라이프사이클에서 단 한번만 호출되도록 보장하게 하는 방식이다.

```solidity
import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";

contract A {
	constructor(address foo) {
		// do something...
	}
}

contract B is Initializable {
    // cannot call initialize more than once due to the `initializer` modifier
    function initialize(
        address foo
    ) public initializer {
        // do something same as contract A contructor code
    }
}
```

위 코드 예시에서 B와 같이 코드를 작성하면 생성자 코드와 동일한 역할을 initialize 함수를 통해 수행할 수 있다. 주의할 점은 반드시 initializer modifier를 적용해야 한다는 것이다.

## 함수 충돌(Function Clashes)

스토리지 충돌과 비슷한 방식으로 함수 레벨에서도 프록시와 로직 컨트랙트 사이에서 충돌이 발생할 수 있다. 프록시 컨트랙트가 어떻게 로직 컨트랙트의 함수를 호출하게 되는지 그 과정을 지난 글(TODO: 링크 추가하기)에서 살펴보았었다. 핵심은 프록시 컨트랙트에 존재하지 않는 함수 식별자를 통해 호출하게 되면, 자연스럽게 fallback 함수로 이어져 delegatecall로 로직 컨트랙트의 함수를 호출하게 되는것이었다.

하지만 만약 프록시 컨트랙트와 로직 컨트랙트에 동일한 함수 식별자가 포함되어 있다면? 예를 들어, 아래와 같은 함수가 두 컨트랙트 모두에 포함되어 있다고 하자.

```solidity
function owner() public view returns (address) {
	return owner;
}
```

컨트랙트의 owner를 지정하고, 이를 확인하는것은 매우 흔한 패턴이다. 프록시, 로직 컨트랙트 모두 컨트랙트 오너가 있는것도 그렇게 이례적인 케이스는 아니다. 

하지만 함수 충돌과 같은 경우 UUPS 패턴을 따르면 발생 가능성이 현저히 낮아진다. 자세한 내용은 아래 UUPS에서 살펴보자.

TODO: 함수 충돌이 UUPS 패턴에서도 발생할 수 있는지? 그리고 아래 내용도 확인필요함. 서로 다른 함수에서도 충돌이 발생할 수 있다는 내용. 지금도 유효한지 확인 필요

*Clashing can also happen among functions with different names. Every function that is part of a contract’s public ABI is identified, at the bytecode level, by a 4-byte identifier. This identifier depends on the name and arity of the function, but since it’s only 4 bytes, there is a possibility that two different functions with different names may end up having the same identifier. The Solidity compiler tracks when this happens within the same contract, but not when the collision happens across different ones, such as between a proxy and its logic contract. Read [this article](https://medium.com/nomic-labs-blog/malicious-backdoors-in-ethereum-proxies-62629adf3357)
 for more info on this.*

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