# [업그레이더블 컨트랙트 씨리즈] Part 1 - 업그레이더블 컨트랙트란?

Created: May 8, 2022 1:26 AM

[업그레이더블 컨트랙트 씨리즈] Part 1 - 업그레이더블 컨트랙트란?

Part 1 - 업그레이더블 컨트랙트란?

Part 2 - 프록시 컨트랙트(Proxy Contract) Deep Dive

Part 2.5 - 복제 가능한 컨트랙트(Cloneable Contract) Deep Dive

Part 3 - Beacon Proxy Contract 딥다이브

Part 4 - Minimal Proxy Contract 딥다이브

Part 5 - Minimal Beacon Proxy Contract 딥다이브

흔히 블록체인에 있는 스마트 컨트랙트의 코드는 절대로 변경할 수 없다는 이야기를 많이 들어왔을것이다. (그것이 “컨트랙트” 즉, 코드로 구현된 계약의 핵심 장점이기도 하다.) 실제로 그 말은 맞다. 하지만, 언제나 그렇듯 개발자들은 원하는 해답을 찾는 사람들이다. 그 해답이 바로 업그레이드 가능한 컨트랙트, 즉 업그레이더블 컨트랙트(Upgredable Contract)이다. 이번 시리즈 글에서는 업그레이더블 컨트랙트와 관련된 모든것들을 해체 분석해보고, 단순히 이해하는것에 그치지 않고 실무에서 바로 사용할 수 있는 수준까지 이해도를 높여보고자 한다.

### 업그레이더블 컨트랙트란?

업그레이더블 컨트랙트는 말 그대로 기능에 대한 업그레이드가 가능한 컨트랙트를 의미한다. 그런데 스마트 컨트랙트의 코드는 절대로 변경할 수 없는데, 어떻게 업그레이드를 한다는 것일까? 그 비밀은 바로 **프록시 패턴(Proxy Pattern)**에 숨겨져 있다. 흔히 컴퓨터 프로그래밍에서 프록시 패턴이라고 한다면, 무언가를 대신해서 이어주는 역할을 해주는 구조를 의미한다. 흔히 접하는 “프록시 서버”의 경우 진짜 서버는 따로 있고, 이 서버와 대신해서 중간에 연결해주는 역할을 프록시 서버가 수행한다. 즉, 대신해서 진짜 서버와 이어주는 역할을 프록시 서버가 담당하게 되는 것이다.

우리가 앞으로 쭉 개발하게 될 스마트 컨트랙트도 이러한 프록시 패턴을 활용하여 업그레이더블 컨트랙트를 구현할 수 있다. 한번 상상을 해보면 쉽게 이해가 된다. 만약 컨트랙트의 코드 실행을 프록시 컨트랙트를 통해 상황에 따라 다른 컨트랙트의 코드를 사용하게 할 수 있다면? 프록시 컨트랙트에 어떤 컨트랙트의 코드를 실행할지만 그 때 그 때 명시한다면 손쉽게 업그레이더블 컨트랙트를 구현할 수 있다. 물론, 지금과 같이 말로만 설명하면 이해가 안되니 간단하게 도식화를 해서 살펴보자.

![Untitled](%5B%E1%84%8B%E1%85%A5%E1%86%B8%E1%84%80%E1%85%B3%E1%84%85%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%83%E1%85%A5%E1%84%87%E1%85%B3%E1%86%AF%20%E1%84%8F%E1%85%A5%E1%86%AB%E1%84%90%E1%85%B3%E1%84%85%E1%85%A2%E1%86%A8%E1%84%90%E1%85%B3%20%E1%84%8A%E1%85%B5%E1%84%85%E1%85%B5%E1%84%8C%E1%85%B3%5D%20Part%201%20-%20%E1%84%8B%E1%85%A5%E1%86%B8%E1%84%80%E1%85%B3%E1%84%85%2064daf0dbdbc549048a5d6acb40e86c15/Untitled.png)

그런데 이런 구조가 도대체 어떻게 가능한것인가? 다른 컨트랙트의 코드를 실행하더라도 이는 다른 컨트랙트의 스토리지(Storage)에 영향을 미칠 뿐, 호출한 컨트랙트의 스토리지에는 영향을 미치지 않는것이 아니었던가.

예를 들어, 다음과 같은 컨트랙트를 작성하여 ERC20 토큰 컨트랙트의 transferFrom()을 호출한다고 해보자. 

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

이 때, `token` 주소의 ERC20 컨트랙트의 transferFrom()을 호출하는것은 당연하게도 해당 컨트랙트의 balance 관련 스토리지를 변경하게 된다. 즉, CallERC20의 스토리지에는 아무런 변화가 없다. 

하지만 만약 transferFrom()을 호출하되, 해당 함수의 로직만 이용하고 관련 스토리지 변경은 CallERC20 컨트랙트에 적용하고 싶다면? 이를 가능하게 하는것이 바로 delegatecall 이다.

Solidity는 다른 컨트랙트를 호출할 때, 두가지 EVM opcode중 하나를 사용할 수 있도록 한다. 바로 call과 delegatecall 이다. call이 바로 우리가 통상적으로 컨트랙트를 호출할 때 사용하는 opcode이다. 바로 위의 CallERC20 예시 또한 내부적으로는 call opcode를 통해 transferFrom()을 호출한다.

사실 이러한 프록시 패턴은 바로 DelegateCall의 존재 때문에 가능하다. 

그렇다면 DelegateCall은 무엇인가? 이를 이해하려면 이더리움의 어카운트 구조를 이해해야 한다.

- DelegateCall 설명
- 이더리움 어카운트 구조 간단하게 설명
- 프록시 패턴에 대한 전반적인 변천사?
    - 프록시 컨트랙트, 비콘 프록시 컨트랙트, 미니멀 프록시 컨트랙트, 미니멀 비콘 프록시 컨트랙트
    

더 읽어보기

- [https://blog.openzeppelin.com/proxy-patterns/](https://blog.openzeppelin.com/proxy-patterns/)
- [https://medium.com/@NipolNIpol/minimal-beacon-proxy-af3cb0c529cf](https://medium.com/@NipolNIpol/minimal-beacon-proxy-af3cb0c529cf)