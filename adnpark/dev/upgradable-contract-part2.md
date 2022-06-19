# [업그레이더블 컨트랙트 씨-리즈] Part 2 - 프록시 컨트랙트 해체 분석하기

Created: 2022년 6월 1일 오전 12:47

## **[업그레이더블 컨트랙트 씨-리즈 목차]**

Part 1 — 업그레이더블 컨트랙트란? 

***Part 2 — 프록시 컨트랙트(Proxy Contract) 해체 분석하기 (이번 글)***

Part 3 — 비콘 프록시 컨트랙트(Beacon Proxy Contract) 해체 분석하기

Part 4 — 미니멀 프록시 컨트랙트(Minimal Proxy Contract) 해체 분석하기

[지난 글](https://medium.com/@aiden.p/%EC%97%85%EA%B7%B8%EB%A0%88%EC%9D%B4%EB%8D%94%EB%B8%94-%EC%BB%A8%ED%8A%B8%EB%9E%99%ED%8A%B8-%EC%94%A8-%EB%A6%AC%EC%A6%88-part-1-%EC%97%85%EA%B7%B8%EB%A0%88%EC%9D%B4%EB%8D%94%EB%B8%94-%EC%BB%A8%ED%8A%B8%EB%9E%99%ED%8A%B8%EB%9E%80-b433225ebf58)에서 업그레이더블 컨트랙트가 왜 등장했고, 어떻게 동작하는지 프록시 패턴을 통해 이해하는 시간을 가졌다. 글의 말미에서 업그레이드시에 실제로 변수가 저장되는 스토리지 영역에 문제가 없을지, 로직 컨트랙트를 계속 변경하면서 새로운 변수가 추가 되더라도 크게 문제가 없을지 등에 대한 의문을 남긴채로 마무리했었다. 

실제로 프록시 컨트랙트를 구현할 때는, 스토리지 충돌(Storage Collision)이라 부르는 위 이슈 뿐만 아니라 여러가지 주의해야 할 사항들이 여러가지 있다. 따라서 이번 글에서는 프록시 컨트랙트를 구현할 때 주의해야 할 여러가지 사항들에 대해 알아보고, 이를 이해하기 위해 필요한 배경지식까지 친절하게 안내해드리고자 한다. 나아가 실무에서 자주 쓰이는 프록시 패턴들에 대해 살펴보고자 한다. 또한 마지막으로 간단한 컨트랙트를 프록시 패턴을 활용하여 업그레이더블 컨트랙트로 구현해보는 실습을 함께 해보자.

먼저 프록시 컨트랙트를 구현할 때 주의해야할 사항들부터 살펴보도록 하자

첫번째 이슈는 스토리지 충돌이다. 프록시 컨트랙트에서 스토리지 충돌이 왜 발생하는지 이해하기 위해서는 이더리움 어카운트의 스토리지에 데이터가 저장되는 방식에 대한 이해가 필요하다. 

### 스토리지 레이아웃(Storage Layout)

<aside>
💡 스토리지 슬롯에 관한 모든것들을 이해하면 당연히 좋겠지만, 이 글의 목적은 업그레이더블 컨트랙트에서 스토리지 충돌이 왜 발생하는지, 그리고 그것을 어떻게 피해갈 수 있는지이다. 이 목적을 염두에 두고 글을 읽어보자.
더 자세한 내용은 “[솔리디티 심화 씨-리즈] 스토리지 레이아웃과 가스 최적화” 글에서 다룰 예정이다.

</aside>

우리가 솔리디티로 컨트랙트를 작성할 때, 여러가지 상태 변수를 선언하고 사용하게 된다. 이 때 상태 변수들이 저장되는 방식을 솔리디티에서는 스토리지 레이아웃이라고 부른다.

스토리지 레이아웃의 가장 기본이 되는 단위는 슬롯으로, 모든 데이터들이 이 슬롯을 기준으로 관리된다.

스토리지 슬롯이 관리되는 규칙은 다음과 같다. 

- 슬롯은 32바이트 크기를 갖는다.
- (일반적으로) 변수가 선언된 순서대로 슬롯에 할당된다.
- 해당 변수 타입의 크기만큼 슬롯을 사용하며, 다음 변수 타입이 기존 슬롯에서 다 포함할 수 없는 경우 다음 슬롯에 할당된다.
    - 단, 동적 배열(Dynamically-sized array)과 매핑(Mapping) 타입은 예외이다.
    - 동적 배열의 경우 배열이 시작해야 하는 슬롯에는 오직 해당 배열의 길이(Length)만 저장한다. 실제 배열 원소들은 배열이 위치한 슬롯 넘버를 `keccak256`으로 해싱한 값이 된다.
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
	items.length = 3;
	items[0] = 12;
	items[1] = 42;
	// allocate mapping values
	values[0] = 100;
	values[1] = 200;
}
```

위와 같은 컨트랙트가 있고, `allocate()` 함수를 호출하여 상태 변수에 값을 저장한다고 할 때, 해당 컨트랙트의 스토리지 레이아웃은 다음과 같다.

![Untitled](%5B%E1%84%8B%E1%85%A5%E1%86%B8%E1%84%80%E1%85%B3%E1%84%85%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%83%E1%85%A5%E1%84%87%E1%85%B3%E1%86%AF%20%E1%84%8F%E1%85%A5%E1%86%AB%E1%84%90%E1%85%B3%E1%84%85%E1%85%A2%E1%86%A8%E1%84%90%E1%85%B3%20%E1%84%8A%E1%85%B5-%E1%84%85%E1%85%B5%E1%84%8C%E1%85%B3%5D%20Part%202%20-%20%E1%84%91%E1%85%B3%E1%84%85%E1%85%A9%E1%86%A8%201f68a3d7b9524dff92a8e5f575aebd43/Untitled.png)

256 비트(32 바이트)크기로 변수가 선언된 순서대로 슬롯에 차곡차곡 할당되는 모습을 확인할 수 있다. 동적 배열이 저장되는 방식, 그리고 매핑의 값들이 매핑이 선언된 순서의 슬롯 넘버와 키값을 해싱한 값의 슬롯 넘버에 저장되는것에 주목해보자. **특히 여기서 매핑 값들이 저장되는 방식이 이후 프록시 패턴에서의 스토리지 충돌을 해결하기 위한 중요한 단초가 된다.**

*여기서 매핑의 데이터가 산발적으로 저장되다보면 데이터간의 충돌이 발생하지 않을까하고 걱정할 수 있지만, 이는 256 비트 해시 함수의 해시 충돌(hash collision) 가능성이 무시 가능하기 때문에 크게 걱정하지 않아도 괜찮다.*

## 스토리지 충돌(Storage Collision)

솔리디티의 스토리지 레이아웃에 대해 이해했으니 이제 스토리지 충돌이 무엇이고, 왜 발생하는지 알아볼 차례다. 

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

![Untitled](%5B%E1%84%8B%E1%85%A5%E1%86%B8%E1%84%80%E1%85%B3%E1%84%85%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%83%E1%85%A5%E1%84%87%E1%85%B3%E1%86%AF%20%E1%84%8F%E1%85%A5%E1%86%AB%E1%84%90%E1%85%B3%E1%84%85%E1%85%A2%E1%86%A8%E1%84%90%E1%85%B3%20%E1%84%8A%E1%85%B5-%E1%84%85%E1%85%B5%E1%84%8C%E1%85%B3%5D%20Part%202%20-%20%E1%84%91%E1%85%B3%E1%84%85%E1%85%A9%E1%86%A8%201f68a3d7b9524dff92a8e5f575aebd43/Untitled%201.png)

프록시 컨트랙트는 `delegatecall`을 활용하여 로직 컨트랙트의 함수를 호출한다. 이는 곧 로직 컨트랙트의 함수를 프록시 컨트랙트의 컨텍스트에서 실행한다는것을 의미하므로, 로직 컨트랙트에서 `foo` 변수를 수정하는 행위는 결국, `implementation` 변수를 수정하게 된다는것을 의미한다. *변수명이 아닌 slot 넘버에 주목하자!*

`implementation` 변수는 로직 컨트랙트의 주소를 저장하는 아주 중요한 부분이다. 하지만, 이 변수에 예상치 못한 다른 값이 할당되게 되면 해당 프록시 컨트랙트는 더 이상 제 기능을 하지 못하게 될 것이다. 이처럼 프록시 컨트랙트에서의 의도치 않은 스토리지 충돌은 컨트랙트에 치명적 결함을 야기할 수 있다.

그렇다면 이는 어떻게 해결할 수 있을까? 프록시 컨트랙트는 어떠한 스토리지 슬롯도 사용하면 안되는걸까? 

이를 해결하기 위한 방법이 바로 [EIP-1967](https://eips.ethereum.org/EIPS/eip-1967)인 Standard Proxy Storage Slots, 즉 표준 프록시 스토리지 슬롯이다. EIP-1967의 아이디어는 매우 단순하다.

스토리지 슬롯을 순차적으로 사용하면 충돌 가능성이 높으니, **슬롯을 랜덤에 가깝게(pseudo-random) 배정**하면 된다는 것이다.

*위 스토리지 레이아웃 파트에서 매핑이 스토리지 슬롯에 저장되는 방식이 어떠했는지 기억하는가? 바로 매핑이 선언된 슬롯의 인덱스 넘버와 매핑의 키값을 해싱하여 밸류가 저장되는 슬롯 위치를 결정하였다. 이와 비슷한 아이디어를 적용해보자는 것이다!*

구체적인 아이디어는 이러하다. 저장하고 싶은 변수의 이름을 keccak256으로 해싱한 후 1을 뺀 값을 슬롯 넘버로 사용하는 것이다. 이 때 주의해야 할 점은 해싱에 사용되는 값을 **절대로 중복해서 사용하지 않아야 한다**는 것이다. 그렇지 않다면 동일하게 스토리지 충돌이 발생하게 된다.

*왜 굳이 1을 빼는가? 에 대해서는 비슷한 질문이 [여기](https://ethereum-magicians.org/t/eip-1967-standard-proxy-storage-slots/3185/7)에 올라온적이 있다. 보안 이슈로 인해 그렇다고 하는데, 확실히 이해하지는 못했다. 혹시 이해한 사람이 있다면 댓글로 꼭 알려주시길!*

```solidity
bytes32 internal constant _IMPLEMENTATION_SLOT = bytes32(uint256(
  keccak256('eip1967.proxy.implementation')) - 1
));
```

그런데 슬롯 넘버는 어떻게 지정하고, 또 해당 슬롯에 있는 값은 어떻게 읽고 쓰는것일까? 이는 오픈제플린의 [ERC1967 관련 컨트랙트](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/proxy/ERC1967/ERC1967Upgrade.sol)와 [StorageSlot 컨트랙트](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/StorageSlot.sol)를 통해 확인할 수 있다.

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

이처럼 EIP-1967을 이용하면 프록시 컨트랙트에서 상태 변수를 사용하면서, 동시에 안전하게 로직 컨트랙트를 사용하고 업그레이드도 무리 없이 진행할 수 있다.

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

로직 컨트랙트를 V1에서 V2로 업그레이드 한다고 가정하자. V2에서는 새롭게 `address baz`라는 상태 변수가 추가 되었다. 그런데, `baz`의 선언 위치를 맨 앞에 두었다. 이렇게 하면 어떤 일이 생길까? 

스토리지 레이아웃을 충분히 이해했다면, 기존 `address foo`의 슬롯에 `address baz`의 슬롯이 할당된다는 것을 알 수 있다. 즉, 스토리지 충돌이 발생하게 되는 것이다. 물론 다른 슬롯의 순서에도 변경이 있었기 때문에, 단순히 `baz` 변수 하나에만 영향을 미치는것은 아니다. 

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

위와 같이 기존 변수의 뒤에 새로운 상태 변수를 선언하면 스토리지 충돌은 발생하지 않는다. 이처럼 상태 변수들의 선언 위치를 하나하나 신경 쓰는것은 상당한 주의를 요한다. 때문에 실제로 업그레이더블 컨트랙트를 작성할 때는 **기존의 로직 컨트랙트를 상속해서 작성**하여 기존의 변수 선언에 변화가 없도록 하는것이 일반적이다.

## 생성자 초기화 코드(Initializing Constructor Code)

프록시 패턴에서는 로직 컨트랙트에서 생성자(Constructor)를 사용할 수 없다. 생성자 함수는 컨트랙트 배포시에만 단 한번 호출되고 런타임 바이트코드에 포함되지 않으므로, 프록시 컨트랙트는 이를 호출할 수 없다. 

그렇다면 프록시 패턴에서는 생성자 함수를 활용할 수 없는것일까? 다행히도 이 역시 피해갈 수 있는 방법이 있다. 소위 약간 짜치는 방식인데, 생성자 함수의 역할을 `initialize()` 함수로 옮기고, 해당 함수가 컨트랙트 라이프사이클에서 단 한번만 호출되도록 보장하게 하는 방식이다.

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

위 코드 예시에서 B와 같이 코드를 작성하면 생성자 코드와 동일한 역할을 `initialize` 함수를 통해 수행할 수 있다. 주의할 점은 반드시 `initializer` modifier를 `initialize` 함수에 적용해야 한다는 것이다.

## 함수 충돌(Function Clashes)

스토리지 충돌과 비슷한 방식으로 함수 레벨에서도 프록시와 로직 컨트랙트 사이에서 충돌이 발생할 수 있다. 프록시 컨트랙트가 어떻게 로직 컨트랙트의 함수를 호출하게 되는지 그 과정을 [지난 글](https://medium.com/@aiden.p/%EC%97%85%EA%B7%B8%EB%A0%88%EC%9D%B4%EB%8D%94%EB%B8%94-%EC%BB%A8%ED%8A%B8%EB%9E%99%ED%8A%B8-%EC%94%A8-%EB%A6%AC%EC%A6%88-part-1-%EC%97%85%EA%B7%B8%EB%A0%88%EC%9D%B4%EB%8D%94%EB%B8%94-%EC%BB%A8%ED%8A%B8%EB%9E%99%ED%8A%B8%EB%9E%80-b433225ebf58)에서 살펴보았었다. 핵심은 프록시 컨트랙트에 존재하지 않는 함수 식별자를 통해 호출하게 되면, 자연스럽게 `fallback` 함수로 이어져 `delegatecall`로 로직 컨트랙트의 함수를 호출하게 되는것이었다.

우리가 주목할점은 프록시 컨트랙트의 기능 구현은 로직 컨트랙트에서 이뤄지는것이 맞지만, 업그레이드 관련 기능을 수행하는 함수는 여전히 필요하다는 것이다. 이 때, 로직 컨트랙트도 동일한 함수를 가지고 있으면 어떻게 되는걸까? 호출자가 프록시 컨트랙트의 업그레이드 함수를 호출하는건지, 로직 컨트랙트의 그것을 호출하는건지 그 의도를 알 수 없게 된다.

```solidity
function upgradeTo(address newImplementation) public view returns (address) {
	// upgrade to newImplementation
}
```

*caption: 동일한 함수 식별자를 갖는 upgradeTo() 함수가 프록시와 로직 컨트랙트 모두에 있다고 생각해보자.*

이 뿐만 아니라 함수 충돌은 생각지도 못한 영역에서 발생할 수 있다. 솔리디티에서 함수 식별자는 함수 시그니처(signature)를 해싱하여 앞의 4바이트만 사용하게 된다. 4바이트가 그렇게 큰 값은 아니기 때문에, 함수명과 파라미터등이 다른 경우에도 얼마든지 충돌이 발생할 수 있다. 그래서 솔리디티 컴파일러는 같은 컨트랙트 내에서 시그니처가 다른데도 불구하고 식별자가 겹치는 경우를 미리 방지한다. 하지만, 프록시와 로직 컨트랙트는 구분된 ‘다른' 컨트랙트이다. 따라서 컴파일 단계에서 이러한 이슈를 미리 방지할 수 없다.

이런 이슈를 해결하기 위해 나온것이 바로 다음에 살펴볼 Transparent 패턴이다.

## Transparent

Transparent 프록시 패턴의 핵심은 두가지이다. 

1. 사용자 어카운트와 어드민 어카운트의 함수 호출 대상 컨트랙트를 구분하는 것
2. 업그레이드 관련 로직을 프록시 컨트랙트에 구현하는 것

1번을 통해 위의 함수 충돌 이슈를 해결하게 된다. 유저 어카운트는 항상 로직 컨트랙트의 함수를 실행하도록 하고, 프록시 컨트랙트 오너는 항상 프록시 컨트랙트의 함수를 실행하도록 한다. 이를 통해 함수 충돌 이슈는 발생하지 않게 된다. 또한, 2번을 통해 프록시 컨트랙트의 오너가 항상 업그레이드를 문제 없이 수행할 수 있도록 보장한다.

Transparent 프록시 패턴은 아래와 같이 구현될 수 있다.

```solidity
contract TransparentProxy {

    function _delegate(address implementation) internal virtual { /*..*/ }

		function implementation() external ifAdmin returns (address implementation_) {
        implementation_ = _implementation();
    }

    fallback() external { /*..*/ } // call _delegate()

		function upgradeTo(address newImplementation) external ifAdmin {
		// check if caller is admin
		// upgrade to newImplementation
		}

		// Makes sure the admin cannot access the fallback function
    function _beforeFallback() internal virtual override {
        require(msg.sender != _getAdmin(), "TransparentUpgradeableProxy: admin cannot fallback to proxy target");
        super._beforeFallback();
    }
}
```

함수 충돌 이슈도 해결했고, 모든게 다 완벽해 보인다. 하지만, 한가지 이슈가 보이는가? `fallback` 함수가 호출될 때마다 `msg.sender`가 `admin` 어카운트인지 아닌지 확인하게 된다. 

우리는 여기서 냄새를 맡아야 한다. 돈이 새는 냄새.

컨트랙트 개발을 할 때 항상 명심해야 하는것. 바로 보안과 가스다. 이 두가지 이슈는 결국 모두 사용자의 자산, 즉 돈과 관련되어 있는 크리티컬 이슈이다. 함수를 호출할 때마다 불필요한 액션이 수반되는것은 치명적인 낭비이다. Transparent 패턴이 제안되는 시점(2018년 무렵)에는 이러한 가스 이슈가 그렇게 크지 않았지만, 두번의 하드포크(이스탄불과 베를린) 이후 `SLOAD` 옵코드와 관련된 가스 비용이 증가하게 되면서 상황은 달라졌다. Transparent 패턴은 함수 충돌을 해결하는데 있어서 상당히 비효율적인 솔루션이 된 것이다.

이를 효율적으로 해결하는 방법으로 개발자 커뮤니티에서 호응을 얻은것이 바로 UUPS 패턴이다. 바로 살펴보도록 하자.

## UUPS(Universal Upgradeable Proxy Standard)

UUPS는 [EIP1822](https://eips.ethereum.org/EIPS/eip-1822) 에서 처음으로 제안된 패턴으로, Transparent 패턴과 달리 업그레이드 로직이 구현체, 즉 로직 컨트랙트에 위치하게 된다. 그래서 프록시 컨트랙트의 구현도 매우 심플해진다. `msg.sender`를 확인하여 프록시 컨트랙트의 함수를 호출할지, 로직 컨트랙트를 호출할지 결정하는 작업은 더 이상 필요 없다. 

아래와 같이 로직 컨트랙트 내부에 업그레이드 함수를 구현하고, 해당 함수 내부에서만 `msg.sender`의 어드민 여부를 확인하면 된다.

```solidity
contract UUPSLogic {
		function upgradeTo(address newImplementation) external virtual onlyProxy {
        _authorizeUpgrade(newImplementation); // check if admin or not
        _upgradeToAndCallUUPS(newImplementation, new bytes(0), false); // upgrade
    }
}
```

UUPS는 Transparent 패턴이 풀고자하는 문제를 더 효율적으로 풀게 되었고, 현재 프록시 패턴에서 가장 흔히 쓰이는 패턴이 되었다. 오픈제플린에서도 [Transparent가 아닌 UUPS 패턴 사용을 권장](https://github.com/OpenZeppelin/openzeppelin-contracts/tree/master/contracts/proxy#transparent-vs-uups-proxies)하고 있다.

UUPS 패턴을 사용할 때 조심해야 할 점은 반드시 업그레이드를 수행할 때, 업그레이드 기능이 포함되어야 한다는 것이다. 그렇지 않을 경우 앞으로 업그레이드를 영원히 진행하지 못하는 이슈가 발생할 수 있다. 따라서, 업그레이드시에 **이를 확인해줄 수 있는 기능이 포함된 검증된 라이브러리를 기반으로 작업하는것을 권장**한다. 이를테면 오픈제플린의 [UUPSUpgradeable](https://docs.openzeppelin.com/contracts/4.x/api/proxy#UUPSUpgradeable1) 컨트랙트를 상속하여 사용하자.

*UUPS 패턴을 사용할 때 소소하지만 큰 장점이 하나 더 있다. 바로 장기적으로 컨트랙트의 탈중앙성을 지킬 수 있다는 것이다. 업그레이드 기능은 아주 편리하지만, 기능이 악용될 여지가 여전히 존재한다는 단점이 있다. 하지만 UUPS는 업그레이드가 영원히 필요하지 않을 때, upgradeTo() 함수를 누구도 호출할 수 없도록 하는 업그레이드를 진행하여 업그레이드 기능을 영원히 봉인할 수 있다!*

## 요약

- 프록시 패턴에서는 스토리지 충돌, 생성자(Constructor) 함수 사용 불가, 함수 충돌과 같은 이슈들이 존재한다.
- 이 이슈들은 다음과 같은 방법으로 해결한다.
    - 스토리지 충돌은 EIP-1967을 사용하여 해결한다.
    - 생성자 함수는 Initialize() 함수로 대체한다.
    - 함수 충돌은 Transparent 패턴으로 해결할 수 있다.
    - 하지만, Transparent 패턴보다 UUPS가 같은 문제를 보다 더 효율적으로 해결할 수 있다.
- 결론: 프록시 컨트랙트 작성하는 베스트 프랙티스는 기본적으로 UUPS 갖다쓰고, 스토리지 충돌이 걱정되는 implementation 주소 같은 변수는 EIP-1967로 해결하고, 로직 컨트랙트의 초기화는 항상 Initialize() 함수로 대체하는 것이다.
- 물론 비콘 프록시라는 놈이 있긴 하다. 이 녀석은 다음 글에서 살펴보도록 하자.

여기까지 프록시 패턴의 기본기를 잘 다져보았다. 다음 글에서는 비콘 프록시라는 새로운 패턴을 배워볼 예정이다. 자꾸 뭐가 많고, 복잡하다는 생각이 많이 들 수 있다. (사실 이 글을 쓰면서 나도 그런 생각이 들었다..) 하지만, 이 모든 것들이 결국 기존의 문제를 더 효율적으로 개선하고, 더 나은 방향으로 가기 위해 고안된것임을 명심하자. 결국 다 우리 좋자고 하는 일이다.