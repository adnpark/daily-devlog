# Soulbound Token

Date: 2022년 6월 22일 오후 11:46

- 소울바운드 토큰에 대한 대략적인 설명이나 원문에 대한 번역은 이미 다른 블로그에 많이 쓰여져 있음
- 나는 구체적인 구현이나 어플리케이션 영역에서 다루는데 더 재밌을듯
- 그리고 현재 사람들이 궁금해할 수 있는 영역일듯?

- 소울 바운드 토큰이 구현이 어렵다 하는데… 왜 구현이 어렵지?
- 소울바운드 토큰을 사고팔지 못하게 하는게 왜 어려운거지? 그냥 넘 쉬울거 같은데?

- 토큰을 특정 어카운트 혹은 ID에 묶는것 (나도 이 생각함)
    - 근데 이 방법은 한계가 있다.
    - 여러 지갑을 쓰는 경우는 어떻게 해야 하는지?
    - 나만해도 여러개 쓰고 있는데…
    - “유저들이 보안과 같은 이유로 자신의 자산을 다른 지갑으로 옮기고 싶어할 수가 있다”
- 그래서 대안이 ENS 주소 자체에 묶으면 어떻겠는가 했던것
    - ENS는 이름같은 개인 아이덴티티가 들어가니까 여기에 귀속시키면 좀 더 낫겠지
- 양도불가능한 NFT를 기존 개인키를 통해 증명할 수 있게 하여 재발급하는 방법
- ID나 프록시에 묶어두는건 어쨌든 그 ID를 사고팔 수 있다는 점에서 엄밀한 의미의 귀속이 아닐수도 있다.
    - 마치 와우 귀속 아이템이 있는 계정을 사고파는것처럼
- Transfer all or none
    - 무조건 다 사거나, 혹은 아예 안사거나하는 방법

- 프라이버시 유출 문제
    - 나의 아이덴티티 관련 정보를 퍼블릭에게 노출시키지 않고, 원할 때 증명할 수 있는가?
    - 영지식 증명? 이 부분 파보면 재밌을듯
    

읽을거리

- [비탈릭 부테린: 귀속 아이템과 NFT에 대한 고찰](https://holycrypto.news/bbs/board.php?bo_table=buzz&wr_id=200)
- ****[Decentralized Society: Finding Web3's Soul](https://deliverypdf.ssrn.com/delivery.php?ID=030073122122018072013002118098026097046038052029001020118029081093021117125027089086120032033122033062107077124069079104101022117009025075093073001085082005011095029088058067118002112007117114005068115065119120094102076119009125105030092118089110119006&EXT=pdf&INDEX=TRUE)****
- ****[Soulbound Token 이해하기 : ② SBT의 활용 가능성](https://ansubin.com/availability-of-soulbound-token/)****
- ****[양도불가능한 NFT: 어떤 때 필요하고 어떻게 구현해야하는가 (feat. soulbound)](https://medium.com/berryfi/%EC%96%91%EB%8F%84%EB%B6%88%EA%B0%80%EB%8A%A5%ED%95%9C-nft-feat-soulbound-611f27a68daf)****
- [EIP 735](https://github.com/ethereum/EIPs/issues/735)
- [EIP 1238](https://github.com/ethereum/EIPs/issues/1238)
- [Proof of Experience](https://medium.com/coinmonks/litepaper-proof-of-experience-da1b76001efa)
- [NFT for Identity](https://medium.com/goldfinch-fi/introducing-unique-identity-uid-the-first-nft-for-identity-830a89207509)
- [CDIP 2- Non-transferable Badges for Maker Ecosystem Activity](https://github.com/makerdao/community/issues/433)