# [NFT 씨리즈] Lazy Minting 만들어보기 Part 2

지난 글에서는 서명과 관련된 부분을 제외하고 레이지 민팅(Lazy Minting)의 핵심 로직들을 구현해보았다. 이제 서명의 검증 및 생성 과정을 직접 구현해볼 차례이다.

서명의 생성 과정은 오프체인(Off-chain)에서 이뤄지고, 검증 과정은 온체인(On-chain)에서 이뤄져야 한다. 서명이 오프체인에서 생성되어 이를 추후에 온체인에서 활용하여 NFT를 민팅하는것이 바로 레이지 민팅이기 때문이다. 따라서 컨트랙트에 직접 구현되는 온체인 서명 검증 과정부터 살펴보자.

```solidity
function redeem(address redeemer, NFTVoucher calldata voucher) public returns (uint256) {
        address signer = _verify(voucher);

        require(signer == voucher.creator, "Signature invalid");
        require(IERC20(voucher.currency).allowance(redeemer, address(this)) >= voucher.price, "Insufficient allowance");

        IERC20(voucher.currency).transferFrom(redeemer, voucher.creator, voucher.price);

        _mint(voucher.creator, voucher.tokenId);
        _setTokenURI(voucher.tokenId, voucher.uri);

        _transfer(voucher.creator, redeemer, voucher.tokenId);

        return voucher.tokenId;
    }
```

redeem 함수에서 가장 먼저 _verify(voucher)를 실행한다.

_verify 함수는 아래와 같이 구현된다.

```solidity
function _verify(NFTVoucher calldata voucher) internal view returns (address) {
        bytes32 digest = _hash(voucher);
        return ECDSA.recover(digest, voucher.signature);
    }
```

먼저 voucher를 해싱한 후 ECDSA 라이브러리를 이용하여 서명을 검증한다. 이더리움은 어카운트의 공개키 암호화를 위해 ECDSA(Elliptic Curve Digital Signature Algorithm)를 사용한다. ECDSA에 대한 자세한 내용은 이 글의 범위 밖이기 때문에 과감하게 생략하자. 우선 잘 모르겠지만, ECDSA 라이브러리를 사용하면 특정 어카운트가 어떤 메세지에 서명을 했는지 아닌지를 검증할 수 있구나라고 이해하고 넘어가도록 하자.

ECDSA.recover(hash, signature) 는 hash 값을 signature로 서명한 어카운트 주소를 리턴한다. 따라서 이렇게 구한 주소가 실제 바우쳐의 creator와 동일한지 체크하면 올바르게 서명을 검증할 수 있다.

```solidity
function _hash(NFTVoucher calldata voucher) internal view returns (bytes32) {
        return _hashTypedDataV4(keccak256(abi.encode(
            keccak256("NFTVoucher(address creator, uint256 tokenId, uint256 price, address currency, string uri"),
            voucher.creator,
            voucher.tokenId,
            voucher.price,
            voucher.currency,
            keccak256(bytes(voucher.uri))
        )));
    }
```

여기서 해시값이 단순히 keccak256을 이용하는것이 아니라 _hashTypedDataV4함수로 한번 더 래핑하는것을 볼 수 있다. 이는 EIP-712와 관련된 것인데, 이와 관련된 내용은 이 글의 범위를 벗어나므로 다음에 별도로 EIP-712를 다루는것으로 하자. 지금은 해싱을 할 때, 데이터의 타입이 명시될 수 있는 형태이구나 정도로 이해하고 넘어가면 된다.

전체 코드와 테스트 코드는 여기서 확인해볼 수 있다. 코드를 참고해서 처음부터 구현해보거나, Repo를 그대로 포크하여 원하는 기능을 입맛에 맞게 추가해보는것도 좋은 연습이 될 것이다. 

- 활용처
    - NFT 마켓플레이스
    - 프로젝트 민팅
        - 자주 쓰이는 방식은 아님

- 응용
    - 바우쳐 관련 데이터를 QR code로 저장, 현금으로 판매 후 redeem

- Recommendation and Caveats