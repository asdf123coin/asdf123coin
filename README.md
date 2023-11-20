- 👋 Hi, I’m @asdf123coin
- 👀 I’m interested in ... Blockchain 
- 🌱 I’m currently learning ... Solidy
- 💞️ I’m looking to collaborate on ... Solana in the future
- 📫 How to reach me ... here :D
// SPDX-License-Identifier: MIT
pragma solidity 0.8.2;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";
import "@openzeppelin/contracts/utils/math/SafeMath.sol";

contract AIUSD is ERC20, ReentrancyGuard, Ownable {
    using SafeMath for uint256;

    uint256 public constant maxSupply = type(uint256).max;
    uint256 public constant fee = 10; // 0.01%
    uint256 public constant pricePrecision = 1e8;
    uint256 public constant oraclePrecision = 1e18;
    uint256 public lastPrice;
    AggregatorV3Interface public priceFeed;
    address public paxGold;

    event Mint(address indexed to, uint256 amount);
    event Burn(address indexed from, uint256 amount);
    event PriceUpdated(uint256 price);

    constructor(address _priceFeed, address _paxGold) ERC20("AIUSD", "AIUSD") {
        priceFeed = AggregatorV3Interface(_priceFeed);
        paxGold = _paxGold;
        lastPrice = getPrice();
    }

    function getPrice() public view returns (uint256) {
        (, int256 price, , ,) = priceFeed.latestRoundData();
        return uint256(price).mul(oraclePrecision).div(pricePrecision);
    }

    function getRatio() public view returns (uint256) {
        uint256 paxGoldBalance = ERC20(paxGold).balanceOf(address(this));
        uint256 totalSupply = totalSupply();
        return paxGoldBalance.mul(pricePrecision).div(totalSupply);
    }

    function mint() external nonReentrant {
        uint256 paxGoldBalance = ERC20(paxGold).balanceOf(address(this));
        uint256 ratio = getRatio();
        uint256 amount = paxGoldBalance.mul(ratio).div(pricePrecision).sub(totalSupply());
        require(amount > 0, "AIUSD: Insufficient collateral");
        _mint(msg.sender, amount);
        emit Mint(msg.sender, amount);
    }

    function burn(uint256 amount) external nonReentrant {
        uint256 ratio = getRatio();
        uint256 paxGoldAmount = amount.mul(ratio).div(pricePrecision);
        require(paxGoldAmount <= ERC20(paxGold).balanceOf(address(this)), "AIUSD: Insufficient collateral");
        _burn(msg.sender, amount);
        emit Burn(msg.sender, amount);
    }

    function updatePrice() external onlyOwner nonReentrant {
        uint256 currentPrice = getPrice();
        require(currentPrice > 0, "AIUSD: Invalid price");

        uint256 delta = currentPrice.mul(pricePrecision).div(lastPrice).sub(pricePrecision);
        uint256 maxDelta = delta.mul(totalSupply()).div(pricePrecision);
        uint256 amount = maxDelta.mul(fee).div(10000);
        uint256 balance = ERC20(paxGold).balanceOf(address(this));
        require(amount <= balance, "AIUSD: Insufficient collateral");

        lastPrice = currentPrice;
        _mint(address(this), amount);
        emit Mint(address(this), amount);
        emit PriceUpdated(currentPrice);
    }

    function withdrawCollateral(uint256 amount) external onlyOwner nonReentrant {
        require(amount > 0, "AIUSD: Invalid amount");
        uint256 balance = ERC20(paxGold).balanceOf(address(this));

    require(amount <= balance, "AIUSD: Insufficient collateral");
    ERC20(paxGold).transfer(msg.sender, amount);
  }
}
