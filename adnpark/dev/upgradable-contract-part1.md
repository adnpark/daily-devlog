# [업그레이더블 컨트랙트 씨리즈] Part 1 - 업그레이더블 컨트랙트란?

Created: May 8, 2022 1:26 AM

[업그레이더블 컨트랙트 씨리즈 목차]

Part 1 - 업그레이더블 컨트랙트란?

Part 2 - 프록시 컨트랙트(Proxy Contract) Deep Dive

Part 2.5 - 복제 가능한 컨트랙트(Cloneable Contract) Deep Dive

Part 3 - Beacon Proxy Contract 딥다이브

Part 4 - Minimal Proxy Contract 딥다이브

Part 5 - Minimal Beacon Proxy Contract 딥다이브

흔히 블록체인에 있는 스마트 컨트랙트의 코드는 절대로 변경할 수 없다는 이야기를 많이 들어왔을것이다. (그것이 “스마트 컨트랙트” 즉, 코드로 구현된 계약의 핵심 장점이기도 하다.) 실제로 그 말은 맞다. 하지만, 코드를 변경할 수 없다는점은 대다수 개발자들에게 매우 치명적으로 들릴 것이다. 만약 코드에 예상치 못한 실수가 있었다면? 만약 이미 배포된 코드에서 치명적인 버그를 발견하게 된다면? (실제로 컨트랙트 개발자들이 가장 두려워하는 순간이다.) 또는 이후에 새로운 기능을 추가하고 싶다면? 어떻게 해야 할지 참 난감하다. 이처럼 코드가 변경 불가능 하다는점은 장점이면서 동시에 단점이 되기도 한다.

하지만, 언제나 그렇듯 개발자들은 원하는 해답을 찾는 사람들이다. “컨트랙트 코드를 어떻게든 변경 가능하게 만드는 방법이 없을까”에 대한 해답이 바로 업그레이드 가능한 컨트랙트, 즉 **업그레이더블 컨트랙트(Upgredable Contract)**이다. 이번 시리즈 글에서는 업그레이더블 컨트랙트와 관련된 모든것들을 해체 분석한 후, 이를 단순히 이해하는것에 그치지 않고 실무에서 실제로 사용할 수 있는 수준까지 이해도를 높여보고자 한다.

### 업그레이더블 컨트랙트란?

업그레이더블 컨트랙트는 말 그대로 기능에 대한 업그레이드가 가능한 컨트랙트를 의미한다. 그런데 스마트 컨트랙트의 코드는 절대로 변경할 수 없다고 했는데, 어떻게 업그레이드를 한다는 것일까? 그 비밀은 바로 **프록시 패턴(Proxy Pattern)**에 숨어 있다. 흔히 컴퓨터 프로그래밍에서 프록시 패턴이라고 한다면, 무언가를 대신해서 이어주는 역할을 해주는 어떠한 구조를 의미한다. 흔히 접하는 “프록시 서버”의 경우 진짜 서버는 따로 있고, 이 서버와 대신해서 중간에 연결해주는 역할을 프록시 서버가 수행한다. 즉, 대신해서 진짜 서버와 이어주는 역할을 프록시 서버가 담당하게 되는 것이다.

우리가 앞으로 쭉 개발하게 될 스마트 컨트랙트도 이러한 프록시 패턴을 활용하여 업그레이더블 컨트랙트를 구현할 수 있다. 한번 상상을 해보자. 만약 컨트랙트의 코드 실행을 프록시 컨트랙트를 통해 상황에 따라 다른 컨트랙트의 코드를 사용할 수 있게 한다면? 프록시 컨트랙트에 어떤 컨트랙트의 코드를 실행할지만 그 때 그 때 명시한다면? 목표로 하는 업그레이더블 컨트랙트를 구현할 수 있지 않을까? 아래 그림을 통해 이를 더 자세하게 살펴보자.

![Untitled](%5B%E1%84%8B%E1%85%A5%E1%86%B8%E1%84%80%E1%85%B3%E1%84%85%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%83%E1%85%A5%E1%84%87%E1%85%B3%E1%86%AF%20%E1%84%8F%E1%85%A5%E1%86%AB%E1%84%90%E1%85%B3%E1%84%85%E1%85%A2%E1%86%A8%E1%84%90%E1%85%B3%20%E1%84%8A%E1%85%B5%E1%84%85%E1%85%B5%E1%84%8C%E1%85%B3%5D%20Part%201%20-%20%E1%84%8B%E1%85%A5%E1%86%B8%E1%84%80%E1%85%B3%E1%84%85%2064daf0dbdbc549048a5d6acb40e86c15/Untitled.png)

프록시 패턴은 위 그림처럼 사용자가 프록시 컨트랙트를 통해 어떠한 함수를 호출하면, 프록시 컨트랙트에 저장되어 있는 주소를 이용하여 로직 컨트랙트를 호출할 수 있도록 한다.

그런데 이런 구조가 도대체 어떻게 가능한것인가? 다른 컨트랙트의 코드를 실행하더라도 이는 다른 컨트랙트의 스토리지(Storage)에 영향을 미칠 뿐, 호출한 컨트랙트의 스토리지에는 영향을 미치지 않았던것 같은데 말이다.

예를 들어, 다음과 같은 컨트랙트를 작성하여 ERC20 토큰 컨트랙트의 `transferFrom()`을 호출한다고 해보자. 

```solidity
import "@openzeppelin/contracts/token/ERC20/IERC20";

contract CallERC20 {
	// ...
	function callTransferFrom(address token, address from, address to, uint256 amount) {
		IERC20(token).transferFrom(from, to, amount);
	}	
	// ...
}
```

이 때, `token` 주소의 ERC20 컨트랙트의 `transferFrom()`을 호출하는것은 당연하게도 `token` 주소 컨트랙트의 `balance` 관련 스토리지를 변경하게 된다. 즉, CallERC20 컨트랙트의 스토리지에는 아무런 변화가 없다. 

### Delegatecall and Proxy Pattern

하지만 만약 transferFrom()을 호출하되, 해당 함수의 로직만 이용하고 관련 스토리지 변경은 CallERC20 컨트랙트에 적용하고 싶다면? 이를 가능하게 하는것이 바로 `delegatecall` 이다.

솔리디티는 다른 컨트랙트를 호출할 때, 크게 두가지 EVM opcode중 하나를 사용할 수 있도록 한다. 바로 `call`과 `delegatecall` 이다. `call`이 바로 우리가 통상적으로 컨트랙트를 호출할 때 사용하는 opcode이다. 바로 위의 CallERC20 예시 또한 내부적으로는 `call` opcode를 통해 `transferFrom()`을 호출한다.

*이외에도 `staticcall` 이라는 친구가 있는데, 상태를 변경하지 않는 call 이라는 의미이다. 지금은 크게 중요하지 않으니 참고만 해두자.*

`delegatecall`은 상당히 특이한 친구인데, 이 opcode는 다른 컨트랙트의 코드를 사용하되, 실행 환경(Context)은 기존 컨트랙트에서 수행될 수 있도록 한다. 아래 다이어그램을 살펴보자.

![Untitled](%5B%E1%84%8B%E1%85%A5%E1%86%B8%E1%84%80%E1%85%B3%E1%84%85%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%83%E1%85%A5%E1%84%87%E1%85%B3%E1%86%AF%20%E1%84%8F%E1%85%A5%E1%86%AB%E1%84%90%E1%85%B3%E1%84%85%E1%85%A2%E1%86%A8%E1%84%90%E1%85%B3%20%E1%84%8A%E1%85%B5%E1%84%85%E1%85%B5%E1%84%8C%E1%85%B3%5D%20Part%201%20-%20%E1%84%8B%E1%85%A5%E1%86%B8%E1%84%80%E1%85%B3%E1%84%85%2064daf0dbdbc549048a5d6acb40e86c15/Untitled%201.png)

A 컨트랙트가 B 컨트랙트를 호출할 때, `delegatecall`을 이용하게 되면 B 컨트랙트의 Code를 사용하지만, Storage는 A 컨트랙트를 사용하게 된다. 트랜잭션 실행의 컨텍스트(Context)가 그대로 유지되는것이 `delegatecall`의 핵심이다.

### 아직 잘 이해가 안된다면

![Untitled](%5B%E1%84%8B%E1%85%A5%E1%86%B8%E1%84%80%E1%85%B3%E1%84%85%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%83%E1%85%A5%E1%84%87%E1%85%B3%E1%86%AF%20%E1%84%8F%E1%85%A5%E1%86%AB%E1%84%90%E1%85%B3%E1%84%85%E1%85%A2%E1%86%A8%E1%84%90%E1%85%B3%20%E1%84%8A%E1%85%B5%E1%84%85%E1%85%B5%E1%84%8C%E1%85%B3%5D%20Part%201%20-%20%E1%84%8B%E1%85%A5%E1%86%B8%E1%84%80%E1%85%B3%E1%84%85%2064daf0dbdbc549048a5d6acb40e86c15/Untitled%202.png)

*이더리움 어카운트 구조를 이미 이해하고 있는 분들은 다음 파트로 넘어가주세요.*

Storage, Code 등 이더리움 어카운트와 관련된 용어가 생소하다면, 이 다이어그램을 참고해보자. 이더리움의 어카운트는 크게 4개의 필드를 갖는다. nonce, balance, storage hash, code hash가 그것이다. nonce는 트랜잭션의 넘버링과 관련된 역할을 수행한다. balance는 해당 어카운트가 보유한 ETH의 잔액을 의미한다. 

중요한 부분은 storage hash와 code hash인데, 이 두가지 모두 컨트랙트 어카운트와 관련되어 있다. (일반적인 사용자 어카운트인 Externally Owned Account(EOA)는 storage와 code가 비어있다.) storage는 말 그대로 저장공간 영역이다. 트랜잭션 실행의 결과로 변경되는 변수들이 저장되는 곳이다. code는 해당 컨트랙트 어카운트의 배포 시점에 결정되는 값인데, 쉽게 생각해서 스마트 컨트랙트 코드가 저장된다고 보면 된다. 이 code hash는 초기화 이후 절대로 변경될 수 없기 때문에, 흔히 스마트 컨트랙트의 코드는 변경될 수 없다는 말을 하는 것이다.

---

다시 돌아가서, `delegatecall`은 다른 컨트랙트 어카운트의 code를 사용하되, storage는 기존 컨트랙트 어카운트의 그것을 사용한다. 이렇게 `delegatecall`을 이용하게 되면 스토리지는 그대로 유지한채로 컨트랙트의 로직이 되는 부분을 상황에 맞게 사용할 수 있게 된다. 프록시 컨트랙트에 어떠한 로직 컨트랙트를 사용할 것인지 주소만 명시해두면 되는것이다.

이론적으로 이해를 해봤으니, 이제 오픈제플린의 프록시 컨트랙트 코드를 함께 살펴보면서 이해도를 더 높여보자.

```solidity
abstract contract Proxy {
    /**
     * @dev Delegates the current call to `implementation`.
     *
     * This function does not return to its internal call site, it will return directly to the external caller.
     */
    function _delegate(address implementation) internal virtual {
        assembly {
            // Copy msg.data. We take full control of memory in this inline assembly
            // block because it will not return to Solidity code. We overwrite the
            // Solidity scratch pad at memory position 0.
            calldatacopy(0, 0, calldatasize())

            // Call the implementation.
            // out and outsize are 0 because we don't know the size yet.
            let result := delegatecall(gas(), implementation, 0, calldatasize(), 0, 0)

            // Copy the returned data.
            returndatacopy(0, 0, returndatasize())

            switch result
            // delegatecall returns 0 on error.
            case 0 {
                revert(0, returndatasize())
            }
            default {
                return(0, returndatasize())
            }
        }
    }
// ...
}
```

Source: [https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/proxy/Proxy.sol](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/proxy/Proxy.sol)

*`assembly {...}` 구문은 솔리디티의 인라인 어셈블리(Inline-Assembly)로, Low level에서 동작을 제어할 수 있도록 도와준다. 솔리디티의 문법과는 사뭇 다른 것들이 많이 보일텐데, 지금은 깊게 살펴보기 보다 큰 흐름만 이해하고 넘어가도록 하자.* 

`_delegate()` 함수는 `delegatecall`을 사용하여 `implementation` 주소의 컨트랙트 로직을 사용할 수 있도록 한다. 먼저 calldata를 통해 (calldata는 말 그대로 call에 필요한 data, 즉, 컨트랙트 함수 실행을 위해 트랜잭션에 포함된 데이터를 의미한다. 지난 글([https://medium.com/@aiden.p/solidity-씨-리즈-은근-헷갈리는-data-location-2690cefb72db](https://medium.com/@aiden.p/solidity-%EC%94%A8-%EB%A6%AC%EC%A6%88-%EC%9D%80%EA%B7%BC-%ED%97%B7%EA%B0%88%EB%A6%AC%EB%8A%94-data-location-2690cefb72db))을 참고해도 좋다.) 함수 호출에 필요한 데이터를 불러 온다. 이 데이터를 통해 `implementation` 컨트랙트 주소로 `delegatecall`을 하고, 성공시에는 결과값을 반환하고 실패할 경우 revert를 하게 된다.

`_delegate()` 함수는 internal 함수이기 때문에 해당 컨트랙트에 포함되거나 상속하는 컨트랙트에서 호출되어야만 한다. 그렇다면 실제로 `_delegate()` 함수는 언제 호출되는지 살펴보자.

```solidity
abstract contract Proxy {
		// ...

		/**
     * @dev This is a virtual function that should be overridden so it returns the address to which the fallback function
     * and {_fallback} should delegate.
     */
    function _implementation() internal view virtual returns (address);

    /**
     * @dev Delegates the current call to the address returned by `_implementation()`.
     *
     * This function does not return to its internal call site, it will return directly to the external caller.
     */
    function _fallback() internal virtual {
        _beforeFallback();
        _delegate(_implementation());
    }

		fallback() external payable virtual {
        _fallback();
    }

		// ...
}
```

`fallback()` 함수를 통해 `_delegate()` 함수가 호출되는것을 확인할 수 있다. `fallback()` 함수란 fallback, 즉, 만일을 위해 대비된 함수로, 주어진 function signature(함수 식별자)와 일치하는 함수가 없거나, 또는 어떠한 calldata도 제공되지 않고, 동시에 `receive` 함수가 없는 경우에 호출된다. 여기서 우리가 활용하고자 하는 케이스는 바로 전자, 즉, 함수 식별자가 일치하지 않는 경우이다.

*함수 식별자(function signature or function selector)에 관한 내용이 궁금하면 이 [영상](https://www.youtube.com/watch?v=Mn4e4w8h6n8)을 참고해보자.*

예를 들어, `function transfer() public returns (bool) {}` 이라는 함수를 호출하고자 할 때, 해당 함수의 식별자가 calldata에 포함되는데, 이 식별자와 일치하는 함수가 해당 컨트랙트에 없으면 `fallback()` 함수가 호출된다. 때문에, 프록시 컨트랙트를 통해 로직 컨트랙트에 구현된 함수들을 호출하면, 해당 함수 식별자와 일치하는 함수가 없기 때문에 `fallback()` 함수가 실행되고, 자연스럽게 `_delegate()` 함수가 실행되는 구조가 된다. 

이제 업그레이드가 필요한 시점에 새로운 로직 컨트랙트를 배포하고, 해당 주소로 `implementation` 주소를 변경해주면 업그레이더블 컨트랙트가 구현된다! 

그런데 말이다.. 로직 컨트랙트를 `delegatecall`로 가져다 쓰는건 이제 알겠는데, 실제로 변수 값은 스토리지에 어떻게 저장되고, 또 업그레이드시에도 정말 문제가 없을까하는 의구심이 들 수 있다. 매우 합리적인 의심이다. 만약 이 글을 읽으면서 이런 생각을 떠올렸다면 스스로 박수를 쳐주자. 

사실 이 부분은 이 글에서 의도적으로 설명하지 않은 부분으로, 실제로 이러한 스토리지를 안전하게 다루기 위해 굉장히 다양한 프록시 패턴이 존재한다. 다음 글 프록시 컨트랙트(Proxy Contract) Deep Dive에서 이 부분에 대해 더 자세히 다뤄보도록 하자. Stay Tuned!

더 읽어보기

- [https://blog.openzeppelin.com/proxy-patterns/](https://blog.openzeppelin.com/proxy-patterns/)
- [https://medium.com/@NipolNIpol/minimal-beacon-proxy-af3cb0c529cf](https://medium.com/@NipolNIpol/minimal-beacon-proxy-af3cb0c529cf)