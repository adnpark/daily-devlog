# [업그레이더블 컨트랙트 씨-리즈] Part 3 - 비콘 프록시 컨트랙트 해체 분석하기

Created: 2022년 6월 21일 오전 2:13

## **[업그레이더블 컨트랙트 씨-리즈 목차]**

Part 1 — [업그레이더블 컨트랙트란?](https://medium.com/@aiden.p/%EC%97%85%EA%B7%B8%EB%A0%88%EC%9D%B4%EB%8D%94%EB%B8%94-%EC%BB%A8%ED%8A%B8%EB%9E%99%ED%8A%B8-%EC%94%A8-%EB%A6%AC%EC%A6%88-part-1-%EC%97%85%EA%B7%B8%EB%A0%88%EC%9D%B4%EB%8D%94%EB%B8%94-%EC%BB%A8%ED%8A%B8%EB%9E%99%ED%8A%B8%EB%9E%80-b433225ebf58) 

Part 2 — [프록시 컨트랙트(Proxy Contract) 해체 분석하기](https://medium.com/@aiden.p/%EC%97%85%EA%B7%B8%EB%A0%88%EC%9D%B4%EB%8D%94%EB%B8%94-%EC%BB%A8%ED%8A%B8%EB%9E%99%ED%8A%B8-%EC%94%A8-%EB%A6%AC%EC%A6%88-part-2-%ED%94%84%EB%A1%9D%EC%8B%9C-%EC%BB%A8%ED%8A%B8%EB%9E%99%ED%8A%B8-%ED%95%B4%EC%B2%B4-%EB%B6%84%EC%84%9D%ED%95%98%EA%B8%B0-95924cb969f0)

***Part 3 — 비콘 프록시 컨트랙트(Beacon Proxy Contract) 해체 분석하기 (이번 글)***

Part 4 — 미니멀 프록시 컨트랙트(Minimal Proxy Contract) 해체 분석하기

## 들어가며

Part 1, 2를 통해 업그레이더블 컨트랙트, 그리고 이를 가능하게 해주는 다양한 프록시 패턴에 대해 살펴보았다. 그리고 나름의 베스트 프랙티스도 학습하였다. (기억이 나지 않는다면 Part 2 요약 부분(TODO: add link)을 다시 복습해보자)

프록시 패턴은 컨트랙트를 업그레이드 가능하게 만들어준다는 장점도 있지만, 한가지 장점이 더 있다. 바로 코드를 재사용 가능하도록 만들어준다는 것이다. 동일한 로직의 재사용은 일반적으로 프로그래밍에서 효율적이라고 여겨진다. 하물며 모든것이 말 그대로 돈인 컨트랙트는 굳이 더 말할것도 없을것이다.

## 컨트랙트를 재사용한다?

![Untitled](%5B%E1%84%8B%E1%85%A5%E1%86%B8%E1%84%80%E1%85%B3%E1%84%85%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%83%E1%85%A5%E1%84%87%E1%85%B3%E1%86%AF%20%E1%84%8F%E1%85%A5%E1%86%AB%E1%84%90%E1%85%B3%E1%84%85%E1%85%A2%E1%86%A8%E1%84%90%E1%85%B3%20%E1%84%8A%E1%85%B5-%E1%84%85%E1%85%B5%E1%84%8C%E1%85%B3%5D%20Part%203%20-%20%E1%84%87%E1%85%B5%E1%84%8F%E1%85%A9%E1%86%AB%201d282e750f9f4e68b145b4e2e6d71f81/Untitled.png)

그렇다면 프록시 패턴은 어떻게 코드를 재사용 가능하도록 만들어주는가? 이를 이해하기 위해서는 프록시 패턴의 핵심 로직을 다시 살펴볼 필요가 있다. 프록시 패턴의 핵심은 로직 컨트랙트에 실행을 위임하는 것에 있다. 여기서 한가지 주목해야 할 점은 로직 컨트랙트에 실행을 위임하는 프록시 컨트랙트는 단 하나만 존재할 이유가 없다는 것이다.

![Untitled](%5B%E1%84%8B%E1%85%A5%E1%86%B8%E1%84%80%E1%85%B3%E1%84%85%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%83%E1%85%A5%E1%84%87%E1%85%B3%E1%86%AF%20%E1%84%8F%E1%85%A5%E1%86%AB%E1%84%90%E1%85%B3%E1%84%85%E1%85%A2%E1%86%A8%E1%84%90%E1%85%B3%20%E1%84%8A%E1%85%B5-%E1%84%85%E1%85%B5%E1%84%8C%E1%85%B3%5D%20Part%203%20-%20%E1%84%87%E1%85%B5%E1%84%8F%E1%85%A9%E1%86%AB%201d282e750f9f4e68b145b4e2e6d71f81/Untitled%201.png)

위 다이어그램처럼 동일한 로직 컨트랙트에 실행을 위임하는 프록시 컨트랙트는 이론상 수 없이 많을 수 있다. 이 점을 잘 활용하면 프록시 패턴을 단순히 업그레이드 목적으로 사용하는것에서 더 나아가, 컨트랙트 배포를 더 효율적으로 수행할 수 있다.

통상적으로 컨트랙트의 배포 비용은 구체적인 컨트랙트의 구현 복잡도에 따라 달라지지만, 대체로 배포시에 필요한 가스는 적게는 수십만에서 많게는 수백만 가스까지 이르게 된다. 때문에 이더리움을 기준으로 컨트랙트의 배포 비용은 매우 높은것이 현실이다. 이 글이 작성되고 있는 시점(2022년 6월 말)에서는 이더의 가격이 많이 내려와 있지만, 가격이 약 3~400만원에 육박할 때는 단순히 컨트랙트 배포만 해도 수백만원(…)이 공중에서 분해되는 기적을 볼 수 있었다. 때문에 개발자의 관점에서 컨트랙트 자체를 효율적으로 구현하는것도 매우 중요하지만, 가능하면 컨트랙트 배포 비용 또한 효율적으로 관리하는것도 중요한 요소이다. 이러한 우리의 니즈를 바로 프록시 패턴이 해결해줄 수 있는 것이다.

## 컨트랙트 10000개 배포하기

![Untitled](%5B%E1%84%8B%E1%85%A5%E1%86%B8%E1%84%80%E1%85%B3%E1%84%85%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%83%E1%85%A5%E1%84%87%E1%85%B3%E1%86%AF%20%E1%84%8F%E1%85%A5%E1%86%AB%E1%84%90%E1%85%B3%E1%84%85%E1%85%A2%E1%86%A8%E1%84%90%E1%85%B3%20%E1%84%8A%E1%85%B5-%E1%84%85%E1%85%B5%E1%84%8C%E1%85%B3%5D%20Part%203%20-%20%E1%84%87%E1%85%B5%E1%84%8F%E1%85%A9%E1%86%AB%201d282e750f9f4e68b145b4e2e6d71f81/Untitled%202.png)

구체적인 사례를 통해 프록시 패턴이 얼마나 많은 컨트랙트 배포 비용을 절감시켜줄 수 있는지 알아보자. 크리에이터 이코노미를 다루는 가상의 프로젝트가 있다고 하자. 이 프로젝트는 크리에이터 10000명을 온보딩 시켰고, 각 크리에이터를 위한 전용 소셜 토큰을 런칭하고자 한다. 즉, 동일한 ERC20 컨트랙트를 10000개나 배포해야 하는 상황인 것이다.

편의상 ERC20 토큰 컨트랙트 배포에 필요한 Gas가 1,500,000 이고, Gas Price는 $0.00005 (ETH Price $1250, 약 40 Gwei 기준)라고 가정해보자. 이 계산에 따르면 토큰 컨트랙트 배포에 필요한 비용은 $75가 된다. 한화로 약 10만원 정도의 나쁘지 않은(?) 비용이다. 하지만 우리는 1개의 ERC20 토큰 컨트랙트가 아닌, 무려 10000개의 컨트랙트를 배포해야 한다. 즉, 전체 비용은 $75가 아닌 $750,000 가 되며, 이는 약 10억원이 넘는 어마무시한 금액이 된다. 

물론 실제로 동일한 컨트랙트를 10000번이나 배포하는게 현실에서 얼마나 많이 발생할지는 모르겠지만, 동일한 컨트랙트를 여러번 배포해야 할 때 필요한 비용이 적지 않다는점은 시사할만 하다. 

![Untitled](%5B%E1%84%8B%E1%85%A5%E1%86%B8%E1%84%80%E1%85%B3%E1%84%85%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%83%E1%85%A5%E1%84%87%E1%85%B3%E1%86%AF%20%E1%84%8F%E1%85%A5%E1%86%AB%E1%84%90%E1%85%B3%E1%84%85%E1%85%A2%E1%86%A8%E1%84%90%E1%85%B3%20%E1%84%8A%E1%85%B5-%E1%84%85%E1%85%B5%E1%84%8C%E1%85%B3%5D%20Part%203%20-%20%E1%84%87%E1%85%B5%E1%84%8F%E1%85%A9%E1%86%AB%201d282e750f9f4e68b145b4e2e6d71f81/Untitled%203.png)

하지만 프록시 패턴을 사용하게 된다면? 이야기는 많이 달라진다. 프록시 컨트랙트는 대체로 로직 컨트랙트의 주소를 관리하는것과 `delegatecall`을 통한 로직 컨트랙트의 호출 부분만 구현되므로 배포시에 필요한 가스가 기존에 비해 상당히 절감되는 편이다. 

TODO: UUPS 프록시 컨트랙트 배포에 필요한 가스 비용 첨부하기

UUPS 프록시 컨트랙트의 경우 배포하는데 필요한 가스 비용은 약 ~~이고, 이를 10000번 배포한다고 가정해보면 총 ~~ 비용이 된다.  10000개의 ERC20 토큰 컨트랙트를 배포하는 동일한 목적을 달성하였지만, 프록시 패턴을 활용할 경우 ~~ 만큼 비용을 절감할 수 있는 것이다.

*사실 여기서 비용을 더 절감할 수 있는 방법이 있다! 자세한 내용은 다음 글 “미니멀 프록시 해체 분석하기”에서 확인해보자.*

## 10000개의 컨트랙트 업그레이드하기

프록시 패턴으로 컨트랙트 배포 비용을 절감하여 10000명의 크리에이터들을 위해 효율적으로 토큰 컨트랙트를 배포해줄 수 있게 되었다. 여기까지는 모든게 다 좋아 보인다. 그런데 한가지 이슈가 생겼다. 커뮤니티의 거버넌스를 통해 토큰을 소각할 수 있게 하자는 안건이 통과된 것이다. 다행히 우리는 토큰 컨트랙트를 업그레이드 가능하도록 프록시 패턴을 사용했다. 따라서 안건을 처리하는게 기술적으로 크게 문제는 없어 보인다. 

그런데 잠깐. 10000개의 컨트랙트를 동일하게 업그레이드 하려면 어떻게 해야 하는걸까?

![Untitled](%5B%E1%84%8B%E1%85%A5%E1%86%B8%E1%84%80%E1%85%B3%E1%84%85%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%83%E1%85%A5%E1%84%87%E1%85%B3%E1%86%AF%20%E1%84%8F%E1%85%A5%E1%86%AB%E1%84%90%E1%85%B3%E1%84%85%E1%85%A2%E1%86%A8%E1%84%90%E1%85%B3%20%E1%84%8A%E1%85%B5-%E1%84%85%E1%85%B5%E1%84%8C%E1%85%B3%5D%20Part%203%20-%20%E1%84%87%E1%85%B5%E1%84%8F%E1%85%A9%E1%86%AB%201d282e750f9f4e68b145b4e2e6d71f81/Untitled%204.png)

불행한 예감은 늘 틀리지 않는다. 모든 컨트랙트를 업그레이드하기 위해서는 위 그림처럼 10000개의 프록시 컨트랙트에서 한땀 한땀 upgradeTo() 함수를 호출해야 한다. 즉 10000번의 업그레이드 실행 트랜잭션이 필요하다. 이런 방식은 사용성을 심각하게 저해하는것은 물론, 가스 비용 또한 폭발적으로 증가시킨다. 또한 만약 컨트랙트의 업그레이드 권한이 개별로 존재할 경우 동시에 업그레이드를 수행하는것이 불가능에 가까워진다.

그렇다면 이런 문제들을 어떻게 개선할 수 있을까? 

## 비콘 프록시(Beacon Proxy)

비콘 프록시 패턴은 프록시와 로직 컨트랙트 사이에 비콘 컨트랙트를 추가로 두는 방식으로 구현된다. 

비콘이라는 단어의 사전적 의미는 다음과 같다.

“봉화나 등대와 같이 위치 정보를 전달하기 위해 어떤 신호를 주기적으로 전송하는 기기”

즉, 비콘 컨트랙트는 말 그대로 프록시 컨트랙트들이 어떤 로직 컨트랙트를 호출해야 하는지를 알려주기 위한 이정표 역할을 하는 것이다.

 아래 그림을 살펴보자.

![Untitled](%5B%E1%84%8B%E1%85%A5%E1%86%B8%E1%84%80%E1%85%B3%E1%84%85%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%83%E1%85%A5%E1%84%87%E1%85%B3%E1%86%AF%20%E1%84%8F%E1%85%A5%E1%86%AB%E1%84%90%E1%85%B3%E1%84%85%E1%85%A2%E1%86%A8%E1%84%90%E1%85%B3%20%E1%84%8A%E1%85%B5-%E1%84%85%E1%85%B5%E1%84%8C%E1%85%B3%5D%20Part%203%20-%20%E1%84%87%E1%85%B5%E1%84%8F%E1%85%A9%E1%86%AB%201d282e750f9f4e68b145b4e2e6d71f81/Untitled%205.png)

프록시 컨트랙트가 직접적으로 로직 컨트랙트를 호출하는것이 아닌 비콘 프록시를 통하게 된다. 이로써 비콘 프록시에 연결된 로직 컨트랙트 주소만 변경해주면 비콘 프록시를 바라보는 수 많은 프록시 컨트랙트들이 동시에 업그레이드가 완료된다.

따라서 위의 10000개 컨트랙트 동시에 업그레이드하기 예시에서 더 이상 10000번의 트랜잭션을, 그것도 한 명이 아닌 수 많은 사람들이 트랜잭션을 전송해야 할 필요 없이 단 한번의 트랜잭션으로 대체 가능해진 것이다.

비콘 프록시 구현 코드를 보면서 이해도를 더 높여보자.

```solidity
contract BeaconProxy is Proxy, ERC1967Upgrade {
		// ...
    function _beacon() internal view virtual returns (address) {
        return _getBeacon();
    }

    function _implementation() internal view virtual override returns (address) {
        return IBeacon(_getBeacon()).implementation();
    }

    function _setBeacon(address beacon, bytes memory data) internal virtual {
        _upgradeBeaconToAndCall(beacon, data, false);
    }
		// ...
}
```

[source](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/proxy/beacon/BeaconProxy.sol)

먼저 오픈제플린의 `BeaconProxy.sol` 컨트랙트부터 살펴보자. 이름에서 알 수 있듯이 “비콘" 방식의 프록시 컨트랙트이다. 가장 눈여겨 볼 점은 `implementation` 컨트랙트 주소를 바로 가져오지 않고, 비콘 컨트랙트를 거쳐서 가져오고 있다는 점이다. 나머지는 일반적인 프록시 컨트랙트와 모두 동일하다. 

```solidity
contract UpgradeableBeacon is IBeacon, Ownable {
		address private _implementation;
		// ...

    function implementation() public view virtual override returns (address) {
        return _implementation;
    }

    function upgradeTo(address newImplementation) public virtual onlyOwner {
        _setImplementation(newImplementation);
        emit Upgraded(newImplementation);
    }

    function _setImplementation(address newImplementation) private {
        require(Address.isContract(newImplementation), "UpgradeableBeacon: implementation is not a contract");
        _implementation = newImplementation;
    }

	// ...
}
```

[Source](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/proxy/beacon/UpgradeableBeacon.sol)

다음은 오픈제플린의 `UpgradeableBeacon.sol` 코드로, 비콘 컨트랙트를 구현한 것이다. 역시 구현이 매우 단순하다. `_implementation` 주소를 비콘 컨트랙트에서 직접 저장한다는 점, 그리고 `upgradeTo()` 함수를 통해 해당 주소를 계속해서 업데이트 할 수 있다는점을 주의깊게 살펴보자.

*`_implementation` 변수로 인한 스토리지 충돌은 걱정하지 않아도 된다. 비콘 컨트랙트는 프록시 컨트랙트의 실행 로직으로 쓰일 컨트랙트가 아니기 때문이다. 비콘 컨트랙트는 오직 로직 컨트랙트의 주소만 열람하는 목적으로 사용된다.*

## 비콘 프록시의 장단점

여기까지만 보면 비콘 프록시는 장점만 존재하는 마치 프록시 패턴의 최종 진화 버전처럼 느껴진다. 실제로 그럴까?

이전 글에서 프록시 패턴 자체가 컨트랙을 한번 거쳐서 실행하는 구조이기 때문에 어느정도 가스 비효율이 발생할 수 밖에 없다고 했었다. 비콘 프록시는 여기에 하나의 레이어를 더 추가한 격이므로, 가스 비효율이 더 크게 증가하게 된다. 

즉, 비콘 프록시 패턴은 대규모 업그레이드의 효율성을 위해 평상시의 트랜잭션 비효율을 어느정도 감수하는것을 선택한 것이다.

그렇다면 비콘 프록시는 언제 사용해야 잘 사용한걸까? 그 답은 매우 간단하다. 

**동일한 로직**을 사용하는 **대규모의** 컨트랙트를 **한번에 업그레이드** 해야하는 경우 사용하면 된다. 위에서 예시로 들었던 크리에이터용 소셜 토큰 컨트랙트 배포가 아주 적합한 예시일 것이다. 

## 요약

- 동일한 로직을 사용하는 컨트랙트를 대규모로 배포하고 동시에 업그레이드 가능하게 하려면 비콘 프록시를 사용하자
- 비콘 프록시는 만능이 아니다. 기존 프록시 패턴에 비해 컨트랙트 호출을 위해 거쳐야 하는 컨트랙트가 하나 더 추가된다. 즉, 트랜잭션 실행을 위한 가스 부담이 증가하게 되므로, 반드시 목적에 맞게 잘 사용해야 한다.
- 이 글에서 다루지는 않았지만, 만약 동일한 로직의 컨트랙트를 대규모로 효율적으로 배포하는것 자체에만 관심이 있다면, 다음 글 미니멀 프록시를 참고해보자.

더 읽어보기

- ****Proxy Patterns For Upgradeability Of Solidity Contracts: Transparent vs UUPS Proxies****

[https://mirror.xyz/0xB38709B8198d147cc9Ff9C133838a044d78B064B/M7oTptQkBGXxox-tk9VJjL66E1V8BUF0GF79MMK4YG0](https://mirror.xyz/0xB38709B8198d147cc9Ff9C133838a044d78B064B/M7oTptQkBGXxox-tk9VJjL66E1V8BUF0GF79MMK4YG0)

[https://mixbytes.io/blog/collisions-solidity-storage-layouts](https://mixbytes.io/blog/collisions-solidity-storage-layouts)

[https://blog.logrocket.com/creating-contract-factory-clone-solidity-smart-contracts/](https://blog.logrocket.com/creating-contract-factory-clone-solidity-smart-contracts/)

- [https://twitter.com/smpalladino/status/1389939156525232130](https://twitter.com/smpalladino/status/1389939156525232130)

[https://blog.openzeppelin.com/the-transparent-proxy-pattern/](https://blog.openzeppelin.com/the-transparent-proxy-pattern/)