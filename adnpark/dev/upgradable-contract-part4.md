# [업그레이더블 컨트랙트 씨-리즈] Part 4 - 미니멀 프록시 컨트랙트 해체 분석하기

Created: 2022년 7월 3일 오후 4:35

## **[업그레이더블 컨트랙트 씨-리즈 목차]**

Part 1 — [업그레이더블 컨트랙트란?](https://medium.com/@aiden.p/%EC%97%85%EA%B7%B8%EB%A0%88%EC%9D%B4%EB%8D%94%EB%B8%94-%EC%BB%A8%ED%8A%B8%EB%9E%99%ED%8A%B8-%EC%94%A8-%EB%A6%AC%EC%A6%88-part-1-%EC%97%85%EA%B7%B8%EB%A0%88%EC%9D%B4%EB%8D%94%EB%B8%94-%EC%BB%A8%ED%8A%B8%EB%9E%99%ED%8A%B8%EB%9E%80-b433225ebf58) 

Part 2 — [프록시 컨트랙트(Proxy Contract) 해체 분석하기](https://medium.com/@aiden.p/%EC%97%85%EA%B7%B8%EB%A0%88%EC%9D%B4%EB%8D%94%EB%B8%94-%EC%BB%A8%ED%8A%B8%EB%9E%99%ED%8A%B8-%EC%94%A8-%EB%A6%AC%EC%A6%88-part-2-%ED%94%84%EB%A1%9D%EC%8B%9C-%EC%BB%A8%ED%8A%B8%EB%9E%99%ED%8A%B8-%ED%95%B4%EC%B2%B4-%EB%B6%84%EC%84%9D%ED%95%98%EA%B8%B0-95924cb969f0)

Part 3 — [비콘 프록시 컨트랙트(Beacon Proxy Contract) 해체 분석하기](https://medium.com/@aiden.p/%EC%97%85%EA%B7%B8%EB%A0%88%EC%9D%B4%EB%8D%94%EB%B8%94-%EC%BB%A8%ED%8A%B8%EB%9E%99%ED%8A%B8-%EC%94%A8-%EB%A6%AC%EC%A6%88-part-3-%EB%B9%84%EC%BD%98-%ED%94%84%EB%A1%9D%EC%8B%9C-%EC%BB%A8%ED%8A%B8%EB%9E%99%ED%8A%B8-%ED%95%B4%EC%B2%B4-%EB%B6%84%EC%84%9D%ED%95%98%EA%B8%B0-82d096a85fcc) 

***Part 4 — 미니멀 프록시 컨트랙트(Minimal Proxy Contract) 해체 분석하기 (이번 글)***

## 들어가며

지난 비콘 프록시 해체 분석하기 글을 마무리 하며 이번 글에서 다룰 주제에 대한 힌트를 남겨놓았다.

“**업그레이드 없이 동일한 로직의 컨트랙트를 대규모로 효율적으로 배포**하는것 자체에만 관심이 있다면, 다음 글 미니멀 프록시를 참고해보자고 했다.”

우리는 프록시 패턴을 계속해서 업그레이더블의 관점에서 바라보고 있었다. 하지만 지난 글 비콘 프록시에서 다뤘듯이, 프록시 패턴은 컨트랙트 로직의 재사용 목적으로 활용될 수 있다. 

완전히 동일한 로직을 갖는 컨트랙트를 여러번 배포해야 하는 경우, 해당 컨트랙트들을 매번 완전히 새롭게 배포할 수도 있다. 하지만, 하나의 로직 컨트랙트를 delegatecall 하는 여러개의 프록시 컨트랙트를 배포하는것으로 동일한 목적을 더 효율적으로 달성할 수 있다. 즉, 프록시 패턴을 통해 컨트랙트 배포 비용을 절약할 수 있는 것이다.

그런데 여기서 배포 비용을 더 최적화 할 수 있는 방법이 없을까? 더 나아가서 업그레이드는 잘 모르겠고, 오로지 컨트랙트의 효율적인 배포만을 최적화 하면 안될까? 

이런 의문에 대한 해답으로 나온것이 바로 오늘 다룰 **EIP-1167: Minimal Proxy Contract, 즉 미니멀 프록시 컨트랙트이다.**

- 미니멀 프록시는 말 그대로 미니멀한, 최소한의 기능만 구현된 프록시 컨트랙트라는 의미다.

- 이러한 목적을 달성하고자 나온것이 바로 **EIP-1167: Minimal Proxy Contract, 즉 미니멀 프록시 컨트랙트이다.**
- 미니멀 프록시는 말 그대로 미니멀한, 최소한의 기능만 구현된 프록시 컨트랙트라는 의미다.
- 그렇다면 어떤것을 최소한으로 구현했다는것일까?
- 업그레이드 기능이 필요없고, 오로지 컨트랙트의 효율적 배포에만 관심이 있다면 굳이 프록시 컨트랙트에 업그레이드와 관련된 코드들이 포함될 필요가 없다.
- 바로 특정 컨트랙트를 delegatecall 하는것만 구현했다.
- 오직 컨트랙트 로직의 재사용만이 목적이라면, 배포되는 모든 컨트랙트가 특정 로직 컨트랙트에 대해서만 delegatecall을 진행하면 될 것이다.
- 업그레이드가 필요 없으므로 로직 컨트랙트의 주소를 관리하거나, 업그레이드하는 함수도 필요하지 않다. 오직 고정된 주소의 컨트랙트로 delegatecall만 실행하는 기능만 필요할 뿐이다.

![Untitled](%5B%E1%84%8B%E1%85%A5%E1%86%B8%E1%84%80%E1%85%B3%E1%84%85%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%83%E1%85%A5%E1%84%87%E1%85%B3%E1%86%AF%20%E1%84%8F%E1%85%A5%E1%86%AB%E1%84%90%E1%85%B3%E1%84%85%E1%85%A2%E1%86%A8%E1%84%90%E1%85%B3%20%E1%84%8A%E1%85%B5-%E1%84%85%E1%85%B5%E1%84%8C%E1%85%B3%5D%20Part%204%20-%20%E1%84%86%E1%85%B5%E1%84%82%E1%85%B5%E1%84%86%203cdff3c95d2f4208a0252446d33d1590/Untitled.png)

- 의사코드를 통해 미니멀 프록시를 간단하게 구현해보자.
- 우리에게 필요한건 단 하나이다. 모든 컨트랙트 실행을 고정된 로직 컨트랙트로 delegatecall을 하고, 그 결과를 반환하는것.
- 여기서 forward 함수가 이를 구현하고 있다.

```solidity
contract PseudoMinimalProxy {
	address public constant IMPLEMENTATION_ADDR = "0xbebebebebebebebebebebebebebebebebebebebe";
	
	function forward() external returns (bytes memory) {
		(bool success, bytes memory data) = IMPLEMENTATION_ADDR.delegatecall(msg.data);
		require(success);

		return data;
	}
}
```

- 이것으로도 충분히 단순하게 구현되지만, 미니멀 프록시는 구현을 더 최적화하기 위해 위 의사 컨트랙트와 동일한 기능을 수행하는 EVM 바이트코드로 구현되었다.
- 실제 미니멀 프록시를 배포하는 코드는 아래와 같다.
- `3d602d80600a3d3981f3363d3d373d3d3d363d73bebebebebebebebebebebebebebebebebebebebe5af43d82803e903d91602b57fd5bf3`
- 무슨 외계어 같아 보이는데, 차근차근 살펴보자.
- 먼저 이를 이해하기 위해서는 EVM 바이트코드가 무엇인지 대략적으로 이해할 필요가 있다.
- 우리가 작성하는 컨트랙트, 즉 솔리디티 코드는 EVM이 이해할 수 있는 EVM 바이트코드로 컴파일되어 이더리움 블록체인에 기록된다.
- 이 때, EVM 바이트코드는 크게 두가지로 구분된다.
- Creation Bytecode(a.k.a Init Code)
    - 컨트랙트 배포를 위해 런타임 바이트코드를 메모리에 복사하여 리턴한다. 생성자 함수의 호출또한 여기에 포함된다.
- Runtime Bytecode(a.k.a Deployed Bytecode)
    - 컨트랙트 배포 후 사용되는 런타임 바이트코드로, 우리가 일반적으로 생각하는 컨트랙트 코드가 바로 이것이다.

- 미니멀 프록시 컨트랙트의 바이트코드도 동일하게 Creation code와 Runtime code로 구분된다.

![Untitled](%5B%E1%84%8B%E1%85%A5%E1%86%B8%E1%84%80%E1%85%B3%E1%84%85%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%83%E1%85%A5%E1%84%87%E1%85%B3%E1%86%AF%20%E1%84%8F%E1%85%A5%E1%86%AB%E1%84%90%E1%85%B3%E1%84%85%E1%85%A2%E1%86%A8%E1%84%90%E1%85%B3%20%E1%84%8A%E1%85%B5-%E1%84%85%E1%85%B5%E1%84%8C%E1%85%B3%5D%20Part%204%20-%20%E1%84%86%E1%85%B5%E1%84%82%E1%85%B5%E1%84%86%203cdff3c95d2f4208a0252446d33d1590/Untitled%201.png)

- 주황색으로 표시된 코드가 Creation code에 해당된다. 지금은 크게 중요하지 않으니 무시하자.
- 우리가 주목해야 할 것은 Runtime code이다.

- Rumtime code를 자세히 살펴보기 전에 미니멀 프록시 컨트랙트에서 런타임에 필요한 동작이 무엇인지 살펴보자.
- 우리가 원하는것은 오직 하나이다. 지정된 컨트랙트 어드레스로 delegatecall을 하고 실행 결과를 return 하는것
- 이를 크게 네가지 세부 동작으로 구분할 수 있다.
1. Calldata 가져오기
2. 지정된 주소로 delegatecall
3. 실행 결과 가져오기
4. 실행 결과에 따라 return 하거나 revert 하기

- calldata가 무엇인지 모르겠다면, 지난 글(TODO: add link)의 설명을 참고해보자.

![Untitled](%5B%E1%84%8B%E1%85%A5%E1%86%B8%E1%84%80%E1%85%B3%E1%84%85%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%83%E1%85%A5%E1%84%87%E1%85%B3%E1%86%AF%20%E1%84%8F%E1%85%A5%E1%86%AB%E1%84%90%E1%85%B3%E1%84%85%E1%85%A2%E1%86%A8%E1%84%90%E1%85%B3%20%E1%84%8A%E1%85%B5-%E1%84%85%E1%85%B5%E1%84%8C%E1%85%B3%5D%20Part%204%20-%20%E1%84%86%E1%85%B5%E1%84%82%E1%85%B5%E1%84%86%203cdff3c95d2f4208a0252446d33d1590/Untitled%202.png)

- 이 동작들을 바이트코드로 구현하면 위 그림과 같다.
- be…be 는 delegatecall의 대상이 될 컨트랙트 주소로, 실제 배포시에는 변경되어야 하는 값이다.

- 마지막으로 Creation code에서 필요한 동작은 간단하다.
- Runtime code를 메모리에 복사한 후 해당 코드를 return 한다.

Creation code 에는 다음 두가지 동작만 필요함

1. Copy the runtime code into memory.
2. Get the code into memory and return it.

이를 구현한것이 오렌지색 부분의 바이트코드이다.

- 만약 바이트코드를 로우레벨에서 더 자세히 이해하고 싶다면 [이 글](https://blog.openzeppelin.com/deep-dive-into-the-minimal-proxy-contract/)을 참고해보자.

1. Get the calldata: `363d3d37`
2. delegate the call: 3d3d3d363d73bebebebebebebebebebebebebebebebebebebebe5af4
3. get the data returned: 3d82803e
4. return or revert: 903d91602b57fd5bf3
5. Creation Bytecode: `3d602d80600a3d3981f3`

```solidity
function clone(address implementation) internal returns (address instance) {
        /// @solidity memory-safe-assembly
        assembly {
            let ptr := mload(0x40)
            mstore(ptr, 0x3d602d80600a3d3981f3363d3d373d3d3d363d73000000000000000000000000)
						/*
						  |             20 bytes                 |
						0x3d602d80600a3d3981f3363d3d373d3d3d363d73000000000000000000000000
						                                          ^ pointer
						*/
            mstore(add(ptr, 0x14), shl(0x60, implementation))
						/*
						  |             20 bytes                 |                20 bytes               |
						0x3d602d80600a3d3981f3363d3d373d3d3d363d73bebebebebebebebebebebebebebebebebebebebe
						                                                                                  ^ pointer
						*/
            mstore(add(ptr, 0x28), 0x5af43d82803e903d91602b57fd5bf30000000000000000000000000000000000)
						/*
						  |             20 bytes                 |                20 bytes               |          15 bytes           |      
						0x3d602d80600a3d3981f3363d3d373d3d3d363d73bebebebebebebebebebebebebebebebebebebebe5af43d82803e903d91602b57fd5bf3
						*/
            instance := create(0, ptr, 0x37)
        }
        require(instance != address(0), "ERC1167: create failed");
}
```

[Clones.sol](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/proxy/Clones.sol)

- 미니멀 프록시 컨트랙트의 바이트코드를 이해했으니, 이를 활용한 실제 컨트랙트 구현 코드를 살펴보자.
- 위 코드는 오픈제플린의 미니멀 프록시 구현체인 Clones.sol이다.
- 이름에서 알 수 있듯이, 컨트랙트를 말 그대로 Clone, 즉 복제하는 기능을 구현한다.
- 알 수 없는 어셈블리 코드가 늘어져있는데, 차근차근 clone() 함수에 대해 살펴보자.
- 먼저 인자로 implementation을 받고 있다, 이는 로직 컨트랙트의 주소를 의미한다.
- let ptr := mload(0x40)
    - 0x40 포인터에 저장된 32 바이트 메모리를 읽어와서 ptr에 할당한다.
    - 0x40 슬롯은 EVM에서 특수한 역할을 담당한다. Free memory pointer로, 할당되어있는 메모리의 마지막 포인터를 의미한다.
- mstore(ptr, 0x3d602d80600a3d3981f3363d3d373d3d3d363d73000000000000000000000000)
    - ptr 위치의 메모리에 32 바이트(0x3d…000)를 저장한다.
    - Creation code 및 1,2번 동작에 대한 코드이다. 클론 대상 컨트랙트 주소 직전까지의 바이트코드이다.
- mstore(add(ptr, 0x14), shl(0x60, implementation))
    - ptr 포인터 위치에서 0x14(20 bytes)를 더한 위치에 implementation 주소를 저장한다.
- mstore(add(ptr, 0x28), 0x5af43d82803e903d91602b57fd5bf30000000000000000000000000000000000)
    - ptr 포인터 위치에서 0x28(40 bytes)를 더한 위치에 나머지 바이트코드를 저장한다.
- instance := create(0, ptr, 0x37)
    - ptr 포인터 위치부터 시작하는 0x37(55 bytes) 사이즈의 코드를 사용하여 새로운 컨트랙트를 배포한다. 새로운 컨트랙트에 0 Ether를 전송한다.
- require(instance != address(0), "ERC1167: create failed");
    - 배포된 컨트랙트의 주소가 zero address가 아닌지 확인한다.
    - 즉, 미니멀 프록시 컨트랙트가 잘 배포되었는지 확인한다.

- Clones.sol에는 clone() 함수 뿐만 아니라 cloneDeterministic() 함수도 있는데, 이는 미니멀 프록시 컨트랙트를 create2 를 이용하여 배포하는것이다. create2는 컨트랙트 주소를 고정하여 배포할 수 있도록 한다. create2는 다음 기회에 다시 자세히 다뤄보자.

## 들어가며

## 요약

- 

더 읽어보기

- [https://blog.openzeppelin.com/deep-dive-into-the-minimal-proxy-contract/](https://blog.openzeppelin.com/deep-dive-into-the-minimal-proxy-contract/)
- [https://medium.com/coinmonks/diving-into-smart-contracts-minimal-proxy-eip-1167-3c4e7f1a41b8](https://medium.com/coinmonks/diving-into-smart-contracts-minimal-proxy-eip-1167-3c4e7f1a41b8)
- [https://blog.originprotocol.com/a-minimal-proxy-in-the-wild-ae3f7b8da990](https://blog.originprotocol.com/a-minimal-proxy-in-the-wild-ae3f7b8da990)
- [https://eips.ethereum.org/EIPS/eip-1167](https://eips.ethereum.org/EIPS/eip-1167)
- [https://www.youtube.com/watch?v=9xqoK2nKkM4](https://www.youtube.com/watch?v=9xqoK2nKkM4)
- [https://medium.com/authereum/bytecode-and-init-code-and-runtime-code-oh-my-7bcd89065904](https://medium.com/authereum/bytecode-and-init-code-and-runtime-code-oh-my-7bcd89065904)
- [https://leftasexercise.com/2021/09/05/a-deep-dive-into-solidity-contract-creation-and-the-init-code/](https://leftasexercise.com/2021/09/05/a-deep-dive-into-solidity-contract-creation-and-the-init-code/)