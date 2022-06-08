# [업그레이더블 컨트랙트 씨-리즈] Part 2 - 프록시 컨트랙트(Proxy Contract) 해체 분석하기

Created: June 1, 2022 12:47 AM

## **[업그레이더블 컨트랙트 씨-리즈 목차]**

Part 1 — 업그레이더블 컨트랙트란? 

***Part 2 — 프록시 컨트랙트(Proxy Contract) 해체 분석하기 (이번 글)***

Part 3 — 복제 가능한 컨트랙트(Cloneable Contract) 해체 분석하기

Part 4 — 비콘 프록시 컨트랙트(Beacon Proxy Contract) 해체 분석하기

Part 5 — 미니멀 프록시 컨트랙트(Minimal Proxy Contract) 해체 분석하기

Part 6 — 미니멀 비콘 프록시 컨트랙트(Minimal Beacon Proxy Contract) 해체 분석하기

지난 글에서 업그레이더블 컨트랙트가 왜 등장했고, 어떻게 동작하는지 프록시 패턴을 통해 이해하는 시간을 가졌다. 글의 말미에서 업그레이드시에 실제로 변수가 저장되는 스토리지 영역에 문제가 없을지, 로직 컨트랙트를 계속 변경하면서 새로운 변수가 추가 되더라도 크게 문제가 없을지 등에 대한 의문을 남긴채로 마무리했었다. 

실제로 프록시 컨트랙트를 구현할 때는, 스토리지 충돌(Storage Collision)이라 부르는 위 이슈 뿐만 아니라 여러가지 주의해야 할 사항들이 있다. 따라서 이번 글에서는 프록시 컨트랙트를 구현할 때 주의해야 할 여러가지 사항들에 대해 알아보고, 이를 이해하기 위해 필요한 배경지식까지 친절하게 안내해드리고자 한다. 나아가 실무에서 자주 쓰이는 프록시 패턴들에 대해 살펴보고자 한다. 또한 마지막으로 간단한 컨트랙트를 프록시 패턴을 활용하여 업그레이더블 컨트랙트로 구현해보는 실습을 함께 해보자.

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