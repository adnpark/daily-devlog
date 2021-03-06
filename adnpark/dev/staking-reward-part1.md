# [솔리디티 실전 씨-리즈] 스테이킹 및 토큰 에미션 컨트랙트 구현해보기

Created: 2022년 6월 3일 오전 2:29

## 들어가며

![Untitled](%5B%E1%84%89%E1%85%A9%E1%86%AF%E1%84%85%E1%85%B5%E1%84%83%E1%85%B5%E1%84%90%E1%85%B5%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%8C%E1%85%A5%E1%86%AB%20%E1%84%8A%E1%85%B5-%E1%84%85%E1%85%B5%E1%84%8C%E1%85%B3%5D%20%E1%84%89%E1%85%B3%E1%84%90%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%8F%E1%85%B5%E1%86%BC%20%E1%84%86%E1%85%B5%E1%86%BE%20%E1%84%90%E1%85%A9%E1%84%8F%E1%85%B3%E1%86%AB%20%E1%84%8B%E1%85%A6%E1%84%86%20b04c18ff85754fa0b0e94967a525c511/Untitled.png)

실제 크립토 프로젝트를 진행하다보면 꼭 한번쯤은 구현하게 되는 기능이 있다. 바로 토큰의 스테이킹과 그 보상 분배 로직을 구현하는 것이다. 토큰 스테이킹 기능은 특히 DeFi 프로젝트들이 급격하게 성장하며 상당히 빠르게 대중화 되었다. 그러다보니 어느정도 베스트 프랙티스가 정립되었는데, 그것이 바로 오늘 우리가 참고할 스시스왑의 MasterChef 컨트랙트이다. 아래 이미지에서 확인할 수 있듯이, 수 많은 토큰 스테이킹 컨트랙트들이 MasterChef 를 포크하는 형태로 구현되었다. 

![Untitled](%5B%E1%84%89%E1%85%A9%E1%86%AF%E1%84%85%E1%85%B5%E1%84%83%E1%85%B5%E1%84%90%E1%85%B5%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%8C%E1%85%A5%E1%86%AB%20%E1%84%8A%E1%85%B5-%E1%84%85%E1%85%B5%E1%84%8C%E1%85%B3%5D%20%E1%84%89%E1%85%B3%E1%84%90%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%8F%E1%85%B5%E1%86%BC%20%E1%84%86%E1%85%B5%E1%86%BE%20%E1%84%90%E1%85%A9%E1%84%8F%E1%85%B3%E1%86%AB%20%E1%84%8B%E1%85%A6%E1%84%86%20b04c18ff85754fa0b0e94967a525c511/Untitled%201.png)

따라서 이번 글에서는 MasterChef 컨트랙트를 참고하여 실제 프로덕션에서 바로 쓰일 수 있을 정도의 스테이킹 컨트랙트를 구현해보고자 한다. 하지만 MasterChef 라는 정답을 미리 보면 조금 재미가 덜하니, 스스로 스시스왑의 개발자가 되었다고 생각해보고 스테이킹 컨트랙트를 밑바닥부터 만들어가보자.

## 스테이킹 보상 계산하기

먼저 기존의 MasterChef 컨트랙트는 여러 LP 토큰을 스테이킹하고 스시토큰을 보상으로 얻을 수 있게 설계되어 있다. 하지만 우리는 요구사항을 조금 더 단순화 시켜보자.

우리가 구현할 스테이킹 컨트랙트의 요구사항은 아래와 같다.

1. 한가지의 토큰만 스테이킹 할 수 있다.
2. 스테이킹 보상은 새로운 리워드 토큰으로 지급된다.
3. 매 블록마다 새로운 보상 토큰이 발행되며, 모든 스테이커들에게 보상으로 분배된다.
4. 각 스테이커가 받는 보상 토큰의 수량은 스테이킹한 토큰의 수량과 그 기간에 비례하여 결정된다.

여기서 핵심은 4번, 즉 스테이커들이 각자 스테이킹한 수량과 기간에 따라 얼마의 보상이 지급되어야 하는지를 계산하는 로직이다. 이를 어떻게 구현해보면 좋을까? 우선 직관의 힘을 빌려 단순하게 생각해보자.

철수라는 스테이커가 있다고 할 때, 철수가 받아야 할 스테이킹 보상은 이렇게 계산해볼 수 있다.

1. 우선 전체 스테이킹 수량에서 철수가 차지하는 **상대적 지분**을 구한다.
2. 여기에 **스테이킹 기간**과 **단위 시간당 보상**을 곱해준다. 

조금 더 이해가 쉽게 예시로 다시 살펴보자.

예를 들어, 새롭게 발행되는 **블록당 토큰 보상이 100개**이고, 스테이킹한 유저는 철수와 영희 두명이 있다고 하자.

![Untitled](%5B%E1%84%89%E1%85%A9%E1%86%AF%E1%84%85%E1%85%B5%E1%84%83%E1%85%B5%E1%84%90%E1%85%B5%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%8C%E1%85%A5%E1%86%AB%20%E1%84%8A%E1%85%B5-%E1%84%85%E1%85%B5%E1%84%8C%E1%85%B3%5D%20%E1%84%89%E1%85%B3%E1%84%90%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%8F%E1%85%B5%E1%86%BC%20%E1%84%86%E1%85%B5%E1%86%BE%20%E1%84%90%E1%85%A9%E1%84%8F%E1%85%B3%E1%86%AB%20%E1%84%8B%E1%85%A6%E1%84%86%20b04c18ff85754fa0b0e94967a525c511/Untitled%202.png)

철수와 영희 모두 동일하게 **5개씩 스테이킹**을 했다면, 철수와 영희의 **상대적 지분은 50%**로 매 블록마다 동일한 상황이다. 따라서 철수와 영희는 모두 동일하게 50개의 스테이킹 보상을 얻게 된다. 

이를 조금 더 일반화하여 수식으로 간단하게 나타내보면 다음과 같다.

r = (s / ts) * t * R

- s = 스테이킹 수량
- ts = 전체 스테이킹 수량
- t = 스테이킹 기간
- R = 블록당 총 스테이킹 보상 수량
- 즉, s / ts 는 특정 스테이커의 지분을 의미한다.

여기까지만 보면 문제가 쉽게 해결된 것 같은 느낌이 든다. 그런데, 여기서 한가지 놓친것이 있다. 바로 스테이킹 토큰 수량과 전체 스테이킹 토큰 수량, 즉 **s와 ts는 고정된 상수가 아니라 변수라는 점**이다.

쉽게 말해, 철수와 영희는 언제든지 더 많은 토큰을 스테이킹하거나, 기존의 스테이킹된 토큰을 출금할 수도 있다. 혹은 민수라는 새로운 친구가 추가로 스테이킹 할수도 있다!

*물론, 민수와 같이 새로운 스테이커의 등장은 기존 스테이커의 스테이킹 수량 변경과 근본적으로 다르지 않으므로, 앞으로의 모든 예시에서 철수와 영희 두명의 스테이커만 존재한다고 가정하고 설명한다.*

예를 들어보자. 만약 영희가 더 많은 보상을 얻고 싶어 3번째 블록에서 10개의 토큰을 추가로 스테이킹했다면 어떻게 될까?

![Untitled](%5B%E1%84%89%E1%85%A9%E1%86%AF%E1%84%85%E1%85%B5%E1%84%83%E1%85%B5%E1%84%90%E1%85%B5%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%8C%E1%85%A5%E1%86%AB%20%E1%84%8A%E1%85%B5-%E1%84%85%E1%85%B5%E1%84%8C%E1%85%B3%5D%20%E1%84%89%E1%85%B3%E1%84%90%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%8F%E1%85%B5%E1%86%BC%20%E1%84%86%E1%85%B5%E1%86%BE%20%E1%84%90%E1%85%A9%E1%84%8F%E1%85%B3%E1%86%AB%20%E1%84%8B%E1%85%A6%E1%84%86%20b04c18ff85754fa0b0e94967a525c511/Untitled%203.png)

3번째 블록부터는 영희의 지분이 75%로 증가하고, 철수의 지분은 25%로 감소하게 된다. 따라서, 철수와 영희의 스테이킹 보상도 다시 계산되어야 한다. 3번째 블록부터 철수가 블록당 얻을 수 있는 토큰 보상은 100 * 0.25 = 25 개가 된다. 당연하게도 영희가 얻을 수 있는 블록당 토큰 보상은 100 * 0.75 = 75 개가 된다.

여기서 다시 앞서 살펴본 수식으로 돌아가보자.

- r = (s / ts) * t * R

우리는 영희의 추가 스테이킹 사례를 통해, 스테이킹 수량의 증감이 있을때마다 위 공식에 새로운 값을 대입해줘야 한다는 것을 알게 되었다. 

예를 들어, 철수가 5번 블록에서 스테이킹 보상을 수령하고 싶다고 할 때, 수령가능한 보상의 양 r(0,4) 은 다음과 같이 계산되어야 한다.

- 0~2번 블록의 스테이킹 보상: r(0,2) = (5 / 10) * 3 * 100 = 150
- 3~4번 블록의 스테이킹 보상: r(3,4) = (5 / 20) * 2 * 100 = 50
- 0~4번 블록의 스테이킹 보상: r(0,4) = r(0,2) + r(3,4) = 150 +50 = 200

마찬가지로 영희의 수령가능한 보상의 양 r(0,4)도 다음과 같이 계산할 수 있다.

- r(0,2) = (5 / 10) * 3 * 100 = 150
- r(3,4) = (15 / 20) * 2 * 100 = 150
- r(0,4) = r(0,2) + r(3,4) = 150 + 150 = 300

이 사례를 통해 확실해진것은 정확한 스테이킹 보상 수량을 구하기 위해서는, **스테이킹 수량에 변동이 있을때마다 이를 반영하여 기록**해야 한다는 것이다.

그럼 이제 이론적인 이해는 충분히 했으니, 직접 컨트랙트로 구현해보자.

## 스테이킹 컨트랙트 구현하기

앞에서 다룬 내용들을 바탕으로 스테이킹 컨트랙트를 아래 코드와 같이 구현해볼 수 있다.

컨트랙트 코드가 조금 장황한데, 하나씩 차례대로 살펴보자.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity >=0.8.4;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/utils/structs/EnumerableSet.sol";
import "./StakeToken.sol";
import "./RewardToken.sol";

import "hardhat/console.sol";

struct Staker {
    uint256 amount; // staking amount
    uint256 rewards; // current cliamable rewards amount
    uint256 lastRewardedBlock; // last block number `rewards` calucated
}

contract NaiveStakingManager {
    using SafeERC20 for StakeToken;
    using SafeERC20 for RewardToken;
    using EnumerableSet for EnumerableSet.AddressSet;

    StakeToken public stakeToken; // Token to be staked
    RewardToken public rewardToken; // Token to be payed as reward

    uint256 public rewardPerBlock; // Reward token emission amount per block
    uint256 public totalStaked; // Total staked tokens

    uint256 public constant SHARE_PRECISION = 1e12;

    EnumerableSet.AddressSet private _stakerList;
    mapping(address => Staker) private _stakers;

    // Events
    event Deposit(address indexed user, uint256 amount);
    event Withdraw(address indexed user, uint256 amount);
    event Claim(address indexed user, uint256 amount);

    constructor(
        address stakeToken_,
        address rewardToken_,
        uint256 rewardPerBlock_
    ) {
        stakeToken = StakeToken(stakeToken_);
        rewardToken = RewardToken(rewardToken_);
        rewardPerBlock = rewardPerBlock_;
    }

    /**
     * @notice must approve stake token first
     * @dev stake tokens
     */
    function deposit(uint256 amount_) public {
        require(amount_ > 0, "should deposit non-zero value");
        Staker storage staker = _stakers[msg.sender];

        // Deposit stake token to this contract
        stakeToken.safeTransferFrom(msg.sender, address(this), amount_);

        // Update rewards of all stakers
        _updateRewards();

        // Update staker info
        if (!_stakerList.contains(msg.sender)) {
            _stakerList.add(msg.sender);
            staker.lastRewardedBlock = block.number;
        }
        staker.amount += amount_;
        totalStaked += amount_;
        emit Deposit(msg.sender, amount_);
    }

    /**
     * @dev withdraw all tokens and claim rewards
     */
    function withdraw() public {
        require(_stakerList.contains(msg.sender), "staker does not exist");

        Staker storage staker = _stakers[msg.sender];
        uint256 withdrawal = staker.amount;
        require(withdrawal > 0, "cannot withdraw zero value");

        // Update rewards of all stakers
        _updateRewards();

        // Claim rewards
        claim();

        // Update staker info
        staker.amount -= withdrawal;
        totalStaked -= withdrawal;
        _stakerList.remove(msg.sender);

        // Withdraw
        stakeToken.safeTransfer(msg.sender, withdrawal);
        emit Withdraw(msg.sender, withdrawal);
    }

    /**
     * @dev claim accumulated rewards
     */
    function claim() public {
        require(_stakerList.contains(msg.sender), "staker does not exist");

        Staker storage staker = _stakers[msg.sender];
        require(staker.amount > 0, "should stake more than 0");

        // Update rewards of all stakers
        _updateRewards();

        // Claim rewards
        uint256 claimed = staker.rewards;
        staker.rewards = 0;
        rewardToken.mint(msg.sender, claimed);
        emit Claim(msg.sender, claimed);
    }

    function updateRewards() public {
        _updateRewards();
    }

    /**
     * @notice must call updateRewards() first to check current pending rewards
     */
    function getPendingRewards(address staker) public view returns (uint256) {
        require(_stakerList.contains(staker), "staker does not exist");
        return _stakers[staker].rewards;
    }

    function getStakingAmount(address staker) public view returns (uint256) {
        require(_stakerList.contains(staker), "staker does not exist");
        return _stakers[staker].amount;
    }

    function getStakingShares(address staker) public view returns (uint256) {
        require(_stakerList.contains(staker), "staker does not exist");
        return (_stakers[staker].amount * SHARE_PRECISION) / totalStaked / SHARE_PRECISION;
    }

    /**
     * @dev loop over all stakers and update their rewards according to relative shares and period
     */
    function _updateRewards() private {
        for (uint256 i = 0; i < _stakerList.length(); i++) {
            Staker storage staker = _stakers[_stakerList.at(i)];
            uint256 stakerShare = (staker.amount * SHARE_PRECISION) / totalStaked;
            uint256 rewardPeriod = block.number - staker.lastRewardedBlock;
            uint256 rewards = (rewardPeriod * rewardPerBlock * stakerShare) / SHARE_PRECISION;
            staker.lastRewardedBlock = block.number;
            staker.rewards += rewards;
        }
    }
}
```

- NaiveStakingManager 컨트랙트는 유저들이 토큰을 스테이킹하여 그 보상으로 새로운 토큰을 받을 수 있게 한다.
- deposit, withdraw, claim 등의 주요 함수들이 존재하며, 각각 토큰을 스테이킹하고, 언스테이킹하고, 보상을 수령하는 기능을 담당한다.
- 그런데 자세히 보면 deposit, withdraw, claim 함수 모두 _updateRewards 함수를 호출하는것을 확인할 수 있다.
- 사실 이 컨트랙트의 핵심 기능은 _updateRewards 함수에 있다고 해도 과언이 아니다.
- 앞에서 우리는 철수와 영희의 스테이킹 시나리오를 통해 스테이킹 토큰 수량에 변동이 있을 때 마다 보상을 산정하는 공식의 변수값이 달라져야 한다는것을 깨달았다.
- 바로 _updateRewards 함수가 바로 이 기능을 담당하고 있다.
- _updateRewards 함수는 모든 스테이커들의 현재 수령 가능한 보상을 계산하여 그 값을 저장한다. 더 정확하게 표현하면, 마지막으로 보상을 계산한 지점부터 현재 시점까지 해당 스테이커의 상대적 지분을 기반으로 받을 수 있는 보상의 양을 계산한다.
    
    ![Untitled](%5B%E1%84%89%E1%85%A9%E1%86%AF%E1%84%85%E1%85%B5%E1%84%83%E1%85%B5%E1%84%90%E1%85%B5%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%8C%E1%85%A5%E1%86%AB%20%E1%84%8A%E1%85%B5-%E1%84%85%E1%85%B5%E1%84%8C%E1%85%B3%5D%20%E1%84%89%E1%85%B3%E1%84%90%E1%85%A6%E1%84%8B%E1%85%B5%E1%84%8F%E1%85%B5%E1%86%BC%20%E1%84%86%E1%85%B5%E1%86%BE%20%E1%84%90%E1%85%A9%E1%84%8F%E1%85%B3%E1%86%AB%20%E1%84%8B%E1%85%A6%E1%84%86%20b04c18ff85754fa0b0e94967a525c511/Untitled%204.png)
    
- 다시 철수와 영희의 예시로 돌아가보자. _updateRewards 함수는 스테이킹 수량에 변동이 있을때마다 호출된다. 즉, 위 예시에서는 총 세번(Block 0, 3, 5) 호출되어 0~5번 블록까지의 보상을 오차 없이 잘 계산하게 된다.
- 정리하면, _updateRewards를 통해 통해 우리는 스테이킹 수량에 변동이 있을때마다 정확하게 보상을 계산할 수 있게 된 것이다.
- 여기까지만 보면 모든 문제가 해결된것처럼 보인다. 요구사항을 충분히 만족하는 형태로 컨트랙트가 잘 구현되었다.
- 하지만, 이 컨트랙트에는 매우 치명적인 단점이 존재한다.
- 스테이킹 수량에 변동이 있을때마다 “모든" 스테이커들을 순회하며 보상을 계속해서 업데이트해야 한다는 점이다.
- 이는 가스 비용 측면에서 매우 큰 비효율을 초래할 수 밖에 없다.
- 단순히 수령 가능한 보상을 보기 위해서도 업데이트를 수동으로 해줘야 하는 불편함은 덤이다.
- 이 때문에 사실 이 컨트랙트에는 앞에 Naive 라는 단어가 붙어있다.
- 그렇다면 이 Naive한 컨트랙트를 효율적인 스테이킹 컨트랙트로 바꾸려면 어떻게 해야할까?
- 만약 모든 스테이커들의 보상을 주기적으로 업데이트 할 필요 없이 스테이킹 보상을 정확하게 계산할 수 있다면 모든 문제가 해결될 것이다.
- 이 지점이 바로 스시스왑의 MasterChef 컨트랙트가 스테이킹 컨트랙트의 베스트 프랙티스인 이유이다.
- 다음 글에서는 본격적으로 스테이킹 컨트랙트의 베스트 프랙티스를 이해하고, 직접 구현하는것까지 다뤄보겠다.

[https://www.nansen.ai/research/all-hail-masterchef-analysing-yield-farming-activity](https://www.nansen.ai/research/all-hail-masterchef-analysing-yield-farming-activity)

## 더 읽어보기

- [https://dev.to/heymarkkop/understanding-sushiswaps-masterchef-staking-rewards-1m6f](https://dev.to/heymarkkop/understanding-sushiswaps-masterchef-staking-rewards-1m6f)
- [https://github.com/sushiswap/sushiswap/blob/archieve/canary/contracts/MasterChefV2.sol](https://github.com/sushiswap/sushiswap/blob/archieve/canary/contracts/MasterChefV2.sol)