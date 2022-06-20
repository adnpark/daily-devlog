# [업그레이더블 컨트랙트 씨-리즈] Part 3 - 비콘 프록시 컨트랙트 해체 분석하기

Created: 2022년 6월 21일 오전 2:13

## **[업그레이더블 컨트랙트 씨-리즈 목차]**

Part 1 — [업그레이더블 컨트랙트란?](https://medium.com/@aiden.p/%EC%97%85%EA%B7%B8%EB%A0%88%EC%9D%B4%EB%8D%94%EB%B8%94-%EC%BB%A8%ED%8A%B8%EB%9E%99%ED%8A%B8-%EC%94%A8-%EB%A6%AC%EC%A6%88-part-1-%EC%97%85%EA%B7%B8%EB%A0%88%EC%9D%B4%EB%8D%94%EB%B8%94-%EC%BB%A8%ED%8A%B8%EB%9E%99%ED%8A%B8%EB%9E%80-b433225ebf58) 

Part 2 — [프록시 컨트랙트(Proxy Contract) 해체 분석하기](https://medium.com/@aiden.p/%EC%97%85%EA%B7%B8%EB%A0%88%EC%9D%B4%EB%8D%94%EB%B8%94-%EC%BB%A8%ED%8A%B8%EB%9E%99%ED%8A%B8-%EC%94%A8-%EB%A6%AC%EC%A6%88-part-2-%ED%94%84%EB%A1%9D%EC%8B%9C-%EC%BB%A8%ED%8A%B8%EB%9E%99%ED%8A%B8-%ED%95%B4%EC%B2%B4-%EB%B6%84%EC%84%9D%ED%95%98%EA%B8%B0-95924cb969f0)

***Part 3 — 비콘 프록시 컨트랙트(Beacon Proxy Contract) 해체 분석하기 (이번 글)***

Part 4 — 미니멀 프록시 컨트랙트(Minimal Proxy Contract) 해체 분석하기

## 들어가며

Part 1, 2를 통해 업그레이더블 컨트랙트, 그리고 이를 가능하게 해주는 다양한 프록시 패턴에 대해 살펴보았다. 그리고 나름의 베스트 프랙티스도 학습하였다. (기억이 나지 않는다면 Part 2 요약 부분을 다시 복습해보자)

아주 기분 좋은 소식이 하나 더 있다. 바로 기존의 비효율을 개선할 수 있는 또 하나의 패턴이 있다는 것이다. 

“*먼저 컨트랙트 배포 비용이 비싸다. → 비용을 줄이고 싶다 → 프록시 패턴을 활용하여 로직 컨트랙트를 재활용하면 줄일 수 있다” 의 흐름으로 작성하면 좋을듯*

- 프록시 패턴의 장점은 업그레이더블에 있기도 하지만, 코드의 재사용성 또한 큰 장점이다.
- 프록시 패턴의 핵심은 로직 컨트랙트에 실행을 위임하는 것이다. 그것이 계속해서 살펴본 delegatecall의 핵심이다.
- 여기서 주목해야 할 점은 로직 컨트랙트에 실행을 위임하는 프록시 컨트랙트는 단 하나만 존재할 이유가 없다는 것이다.
- 이 점을 잘 활용하면 프록시 패턴을 업그레이더블 컨트랙트 뿐만 아니라, 컨트랙트 배포의 관점에서도 효율성을 배가시킬 수 있다.
- 컨트랙트의 구현 복잡도에 따라 다르지만, 대체로 컨트랙트를 배포할 때 상당히 많은 가스가 소모되고 이로 인해 배포 비용이 매우 높다.
- 이 글이 작성되고 있는 시점은 이더의 가격이 많이 내려와 있지만, 가격이 약 3~400 만원에 육박할 때는 컨트랙트만 배포만 해도 수백만원(…)이 날라가는 기적을 볼 수 있었다.
- 때문에 개발자 입장에서 컨트랙트 내부의 동작을 효율적으로 구현하는것도 중요하지만, 컨트랙트 배포 비용또한 가급적 줄일 수 있으면 좋다.
- 이러한 니즈를 프록시 패턴이 해결해줄 수 있는데, 바로 동일한 로직 컨트랙트를 재사용하는 것이다.
- 예를 들어, 동일한 ERC20 토큰 컨트랙트를 10000개 배포해야 한다고 가정해보자.
- 그리고 편의상 토큰 컨트랙트 배포에 필요한 Gas는 500000, Gas Price는 $0.7라고 가정해보자.
- ERC20 컨트랙트 배포에 필요한 비용은 무려 350,000
- 
- TODO: 그림 추가 - 다수의 프록시 컨트랙트가 동일한 로직 컨트랙트를 가르키는 모습

더 읽어보기

- ****Proxy Patterns For Upgradeability Of Solidity Contracts: Transparent vs UUPS Proxies****

[https://mirror.xyz/0xB38709B8198d147cc9Ff9C133838a044d78B064B/M7oTptQkBGXxox-tk9VJjL66E1V8BUF0GF79MMK4YG0](https://mirror.xyz/0xB38709B8198d147cc9Ff9C133838a044d78B064B/M7oTptQkBGXxox-tk9VJjL66E1V8BUF0GF79MMK4YG0)

[https://mixbytes.io/blog/collisions-solidity-storage-layouts](https://mixbytes.io/blog/collisions-solidity-storage-layouts)

[https://blog.logrocket.com/creating-contract-factory-clone-solidity-smart-contracts/](https://blog.logrocket.com/creating-contract-factory-clone-solidity-smart-contracts/)

- [https://twitter.com/smpalladino/status/1389939156525232130](https://twitter.com/smpalladino/status/1389939156525232130)

[https://blog.openzeppelin.com/the-transparent-proxy-pattern/](https://blog.openzeppelin.com/the-transparent-proxy-pattern/)