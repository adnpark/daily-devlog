# [업그레이더블 컨트랙트 씨-리즈] Part 4 - 미니멀 프록시 컨트랙트 해체 분석하기

Created: 2022년 7월 3일 오후 4:35

## **[업그레이더블 컨트랙트 씨-리즈 목차]**

Part 1 — [업그레이더블 컨트랙트란?](https://medium.com/@aiden.p/%EC%97%85%EA%B7%B8%EB%A0%88%EC%9D%B4%EB%8D%94%EB%B8%94-%EC%BB%A8%ED%8A%B8%EB%9E%99%ED%8A%B8-%EC%94%A8-%EB%A6%AC%EC%A6%88-part-1-%EC%97%85%EA%B7%B8%EB%A0%88%EC%9D%B4%EB%8D%94%EB%B8%94-%EC%BB%A8%ED%8A%B8%EB%9E%99%ED%8A%B8%EB%9E%80-b433225ebf58) 

Part 2 — [프록시 컨트랙트(Proxy Contract) 해체 분석하기](https://medium.com/@aiden.p/%EC%97%85%EA%B7%B8%EB%A0%88%EC%9D%B4%EB%8D%94%EB%B8%94-%EC%BB%A8%ED%8A%B8%EB%9E%99%ED%8A%B8-%EC%94%A8-%EB%A6%AC%EC%A6%88-part-2-%ED%94%84%EB%A1%9D%EC%8B%9C-%EC%BB%A8%ED%8A%B8%EB%9E%99%ED%8A%B8-%ED%95%B4%EC%B2%B4-%EB%B6%84%EC%84%9D%ED%95%98%EA%B8%B0-95924cb969f0)

Part 3 — [비콘 프록시 컨트랙트(Beacon Proxy Contract) 해체 분석하기](https://medium.com/@aiden.p/%EC%97%85%EA%B7%B8%EB%A0%88%EC%9D%B4%EB%8D%94%EB%B8%94-%EC%BB%A8%ED%8A%B8%EB%9E%99%ED%8A%B8-%EC%94%A8-%EB%A6%AC%EC%A6%88-part-3-%EB%B9%84%EC%BD%98-%ED%94%84%EB%A1%9D%EC%8B%9C-%EC%BB%A8%ED%8A%B8%EB%9E%99%ED%8A%B8-%ED%95%B4%EC%B2%B4-%EB%B6%84%EC%84%9D%ED%95%98%EA%B8%B0-82d096a85fcc) 

***Part 4 — 미니멀 프록시 컨트랙트(Minimal Proxy Contract) 해체 분석하기 (이번 글)***

- 비콘 프록시 글을 마무리 하면서 이번 글에서 다룰 주제에 대한 힌트를 남겨놓았다
- **업그레이드 없이 동일한 로직의 컨트랙트를 대규모로 효율적으로 배포**하는것 자체에만 관심이 있다면, 다음 글 미니멀 프록시를 참고해보자고 했다.
- 이게 어떤 의미일까?
- 우리는 계속해서 업그레이더블의 관점에서 프록시 패턴을 바라보고 있다.
- 하지만, 지난 글 비콘 프록시에서 다뤘듯 프록시 패턴은 컨트랙트 로직의 재사용의 목적으로도 사용될 수 있다.
- 동일한 컨트랙트를 여러번 배포해야 할 경우, 복잡한 로직 컨트랙트를 매번 동일하게 배포하는것이 아니라, 해당 로직 컨트랙트에 실행을 위임하는 여러 프록시 컨트랙트를 배포하면 컨트랙트 배포 비용을 대폭 절감할 수 있다.
- 여기까지도 괜찮긴한데, 컨트랙트 배포 비용을 더 줄일 수 있는 방법은 없을까?
- 업그레이드 기능이 필요없고, 오로지 컨트랙트의 효율적 배포에만 관심이 있다면 굳이 프록시 컨트랙트에 업그레이드와 관련된 코드들이 포함될 필요가 없다.
- 이러한 목적을 달성하고자 나온것이 바로 **EIP-1167: Minimal Proxy Contract, 즉 미니멀 프록시 컨트랙트이다.**
- 미니멀 프록시는 말 그대로 미니멀한, 최소한의 기능만 구현된 프록시 컨트랙트라는 의미다.
- 

- 미니멀 프록시 구현 코드 분석
- EVM 바이트코드로 이루어져 있다.

## 들어가며

## 요약

- 동일한 로직을 사용하는 컨트랙트를 대규모로 배포하고 동시에 업그레이드 가능하게 하려면 비콘 프록시를 사용하자
- 비콘 프록시는 만능이 아니다. 기존 프록시 패턴에 비해 컨트랙트 호출을 위해 거쳐야 하는 컨트랙트가 하나 더 추가된다. 즉, 트랜잭션 실행을 위한 가스 부담이 증가하게 되므로, 반드시 목적에 맞게 잘 사용해야 한다.
- 이 글에서 다루지는 않았지만, 만약 동일한 로직의 컨트랙트를 대규모로 효율적으로 배포하는것 자체에만 관심이 있다면, 다음 글 미니멀 프록시를 참고해보자.

더 읽어보기

- [https://blog.openzeppelin.com/deep-dive-into-the-minimal-proxy-contract/](https://blog.openzeppelin.com/deep-dive-into-the-minimal-proxy-contract/)
- [https://medium.com/coinmonks/diving-into-smart-contracts-minimal-proxy-eip-1167-3c4e7f1a41b8](https://medium.com/coinmonks/diving-into-smart-contracts-minimal-proxy-eip-1167-3c4e7f1a41b8)
- [https://blog.originprotocol.com/a-minimal-proxy-in-the-wild-ae3f7b8da990](https://blog.originprotocol.com/a-minimal-proxy-in-the-wild-ae3f7b8da990)
- [https://eips.ethereum.org/EIPS/eip-1167](https://eips.ethereum.org/EIPS/eip-1167)
- [https://www.youtube.com/watch?v=9xqoK2nKkM4](https://www.youtube.com/watch?v=9xqoK2nKkM4)