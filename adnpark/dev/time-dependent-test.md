# 컨트랙트 테스트에서 시간 조작하는법

Created: 2022년 7월 29일 오전 2:16

- 컨트랙트 테스트 작업을 하다보면 시간을 변경해야하는 경우가 빈번하다.
- 예를 들어, NFT 보유자를 대상으로 일정 기간 후 Redeem 하는 이벤트를 진행해야 한다면? 특정 시간 후 NFT가 제대로 Redeem 될 수 있는지 테스트할 수 있어야 할 것이다.
- 뿐만 아니라, 스테이킹 보상과 같이 불특정 다수의 유저에게 보상을 지급해야 하는 컨트랙트의 경우, 제대로 테스트하기 위해서는 여러가지 케이스를 두고 시간을 변경해가며 의도한대로 동작하는지 확인해야 할 것이다.

- 이처럼 시간을 조작해야 하는 테스트의 경우 크게 두가지 케이스로 나뉜다.
- 첫째, 블록 타임을 조정해야 하는 경우
- 둘째, 블록 넘버를 조정해야 하는 경우

- 이 글에서는 이 두가지 케이스를 테스트 환경에서 효과적으로 제어할 수 있는 방법들에 대해 알아본다.
- 이 글의 테스트 환경은 Hardhat을 기준으로 작성된다.

- [https://ethereum.stackexchange.com/questions/98935/modify-block-number-when-testing-with-hardhat](https://ethereum.stackexchange.com/questions/98935/modify-block-number-when-testing-with-hardhat)
- [https://hardhat.org/hardhat-network/docs/reference#hardhat_mine](https://hardhat.org/hardhat-network/docs/reference#hardhat_mine)
- [https://ethereum.stackexchange.com/questions/86633/time-dependent-tests-with-hardhat](https://ethereum.stackexchange.com/questions/86633/time-dependent-tests-with-hardhat)