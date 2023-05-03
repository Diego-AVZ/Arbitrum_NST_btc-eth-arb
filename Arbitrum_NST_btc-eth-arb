// SPDX-License-Identifier: MIT

/**
 * @author : Nebula.fi
 * wBTC-wETH
 */

pragma solidity ^0.8.7;
//modificar IMPORT
import "../../utils/ChainId.sol";
import "./NGISplitter.sol";
import "hardhat/console.sol";
import "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/token/ERC20/ERC20Upgradeable.sol";
import "@openzeppelin/contracts-upgradeable/security/PausableUpgradeable.sol";

contract GenesisIndex is ERC20Upgradeable, OwnableUpgradeable, PausableUpgradeable, ChainId, NGISplitter {
    event Mint(address indexed from, uint256 wbtcIn, uint256 wethIn, uint256 indexed amount);
    event Burn(address indexed from, uint256 usdcIn, uint256 indexed amount);

    /// @custom:oz-upgrades-unsafe-allow constructor
    constructor () {
        _disableInitializers();
    }



    function initialize() public initializer{
        __ERC20_init("Nebula Synthetic Token btc/eth/arb - Arbitrum", "NST_arbitrum");
        __Ownable_init();
        __Pausable_init();
        tokens = [
            0xFF970A61A04b1cA14834A43f5dE4533eBDDB5CC8, //[0] => USDC
            0x2f2a2543B76A4166549F7aaB2e75Bef0aefC5B0f, //[1] => wBTC
            0x82aF49447D8a07e3bd95BD0d56f35241523fBab1, // [2] => wETH
            0x912CE59144191C1204E64559FE8253a0e49E6548 // [3] => ARB 
        ];
        multipliers = [1e12, 1e10, 1, 1];
        marketCapWeigth = [0, 4500, 4000, 1500];
        uniV3 = ISwapRouter(0x4C60051384bd2d3C01bfc845Cf5F4b44bcbE9de5); // UNISWAP_universal_router
        quick = IUniswapV2Router02(0xaBBc5F99639c9B6bCb58544ddf04EFA6802F4064); //GMX

        addressRouting = [0x4C60051384bd2d3C01bfc845Cf5F4b44bcbE9de5, 0xaBBc5F99639c9B6bCb58544ddf04EFA6802F4064];

        priceFeeds = [
            AggregatorV3Interface(0x50834f3163758fcc1df9973b6e91f0f0f0434ad3), // USDC
            AggregatorV3Interface(0x6ce185860a4963106506c203335a2910413708e9), // wBTC
            AggregatorV3Interface(0x639fe6ab55c921f74e7fac1ee960c0b6293ba612), // wETH
            AggregatorV3Interface(0xb2a824043730fe05f3da2efafa1cbbe83fa548d6)  // ARB
        ];
       

    }

    /**
     * @notice Returns the price of the index
     * wETH/usdc * 0.4 + wBTC/usdc * 0.45 + ARB/usdc * 0.15
     */
    function getVirtualPrice() public view returns (uint256) {
        return (((getLatestPrice(1) * 4500) / 10000) + ((getLatestPrice(2) * 4000) / 10000) + ((getLatestPrice(3) * 1500) / 10000));
    }

    /**
     * @notice function to buy 74% wBTC and 26% wETH with usdc
     * @param tokenIn : the token to deposit, must be a component of the index(0,1,2)
     * @param amountIn : token amount to deposit
     * @param optimization: level of slippage optimization, from 0 to 4 
     * @param recipient : recipient of the NGI tokens
     * @return shares : amount of minted tokens
     */
    function deposit(uint8 tokenIn, uint256 amountIn, uint8 optimization, address recipient)
        public
        whenNotPaused
        returns (uint256 shares)
    {
        require(optimization < 5, "optimization >= 5");
        require(tokenIn < 4, "token >=4");
        require(amountIn > 0, "dx=0");
        uint256 dywBtc;
        uint256 dywEth;
        uint256 dywArb;
        uint8 i = tokenIn;

        TransferHelper.safeTransferFrom(tokens[i], msg.sender, address(this), amountIn);

        (uint256 amountForBtc, uint256 amountForEth) = ((amountIn * 7400) / 10000, (amountIn * 2600) / 10000);

        approveAMM(i, amountIn, optimization + 1);
        dywBtc = swapWithParams(i, 1, amountForBtc, optimization + 1);
        dywEth = swapWithParams(i, 2, amountForEth, optimization + 1);
        dywArb = swapWithParams(i, 3, amountForArb, optimization + 1);

        _mint(
            recipient,
            shares = ((dywBtc * multipliers[1] * getLatestPrice(1)) + (dywEth * multipliers[2] * getLatestPrice(2)) + (dywArb * multipliers[3] * getLatestPrice(3)))
                / getVirtualPrice()
        );
        emit Mint(recipient, dywBtc, dywEth, dywArb, shares);
    }

    /**
     * @notice function to buy 45% wBTC + 40% wETH + 15% ARB with usdc choosing a custom AMM split, previously calculated off-chain
     * @param tokenIn : the token to deposit, must be a component of the index(0,1,2,3)
     * @param amountIn : amount of the token to deposit
     * @param percentagesWBTCSplit : percentages of the token to exchange in each dex to buy WBTC
     * @param percentagesWETHSplit : percentages of the token to exchange in each dex to buy WETH
      * @param percentagesARBSplit : percentages of the token to exchange in each dex to buy ARB
     * @param recipient : recipient of the NST
     * @return shares : amount of minted tokens
     */

    function depositCustom(
        uint8 tokenIn,
        uint256 amountIn,
        uint16[5] calldata percentagesWBTCSplit,
        uint16[5] calldata percentagesWETHSplit,
        uint16[5] calldata percentagesARBSplit,
        address recipient
    ) external whenNotPaused returns (uint256 shares) {
        uint8 i = tokenIn;
        require(i < 4);
        require(amountIn > 0, "dx=0");
        require(_getTotal(percentagesWBTCSplit) == 10000 && _getTotal(percentagesWETHSplit) == 10000 && _getTotal(percentagesARBSplit) == 10000, "!=100%");
        TransferHelper.safeTransferFrom(tokens[i], msg.sender, address(this), amountIn);

        approveAMM(i, amountIn, 5);
        uint256 amountForBtc = amountIn * 4500 / 10000;
        uint256 amountForEth = amountIn * 4000 / 10000;
        uint256 amountForArb = amountIn * 1500 / 10000;
        uint256[5] memory splitsForBtc;
        uint256[5] memory splitsForEth;
        uint256[5] memory splitsForArb;

        for (uint256 index = 0; index < 5;) {
            splitsForBtc[index] = amountForBtc * percentagesWBTCSplit[index] / 10000;
            splitsForEth[index] = amountForEth * percentagesWETHSplit[index] / 10000;
            splitsForArb[index] = amountForArb * percentagesArbSplit[index] / 10000;
            unchecked {
                ++index;
            }
        }

        uint256 dywBtc = swapWithParamsCustom(i, 1, splitsForBtc);
        uint256 dywEth = swapWithParamsCustom(i, 2, splitsForEth);
        uint256 dywArb = swapWithParamsCustom(i, 3, splitsForArb);

        _mint(
            recipient,
            shares = (dywBtc * multipliers[1] * getLatestPrice(1) + dywEth * multipliers[2] * getLatestPrice(2) + dywArb * multipliers[3] * getLatestPrice(3))
                / getVirtualPrice()
        );
        emit Mint(recipient, dywBtc, dywEth, dywArb,shares);
    }

    /**
     * @notice Function to liquidate wETH and wBTC positions for usdc
     * @param ngiIn : the number of indexed tokens to burn 
     * @param optimization: level of slippage optimization, from 0 to 4
     * @param recipient : recipient of the USDC
     * @return usdcOut : final usdc amount to withdraw after slippage and fees
     */
    function withdrawUsdc(uint256 NSTin, uint8 optimization, address recipient)
        external
        whenNotPaused
        returns (uint256 usdcOut)
    {
        require(NSTin > 0, "dx=0");
        require(optimization < 5, "optimization >= 5");

        uint256 balanceWBtc = IERC20(tokens[1]).balanceOf(address(this));
        uint256 balanceWEth = IERC20(tokens[2]).balanceOf(address(this));
        uint256 balanceArb = IERC20(tokens[3]).balanceOf(address(this));
        uint256 wBtcIn = balanceWBtc * NSTin / totalSupply();
        uint256 wEthIn = balanceWEth * NSTin / totalSupply();
        uint256 ArbIn = balanceArb * NSTin / totalSupply();

        _burn(msg.sender, NSTin);
        approveAMM(1, wBtcIn, optimization + 1);
        approveAMM(2, wEthIn, optimization + 1);
        approveAMM(3, ArbIn, optimization + 1);
        TransferHelper.safeTransfer(
            tokens[0],
            recipient,
            usdcOut = swapWithParams(1, 0, wBtcIn, optimization + 1) + swapWithParams(2, 0, wEthIn, optimization + 1) + swapWithParams(3, 0, ArbIn, optimization + 1)
        );
        emit Burn(recipient, usdcOut, NSTin);
    }

    /**
     * @notice Function to liquidate wETH and wBTC positions for usdc
     * @param NSTin : the number of indexed tokens to burn 
     * @param percentagesWBTCSplit : percentages of the token to exchange in each dex to buy WBTC
     * @param percentagesWETHSplit : percentages of the token to exchange in each dex to buy WETH
     * @param recipient : recipient of the USDC
     * @return usdcOut : final usdc amount to withdraw after slippage and fees
     */

    function withdrawUsdcCustom(
        uint256 NSTin,
        uint16[5] calldata percentagesWBTCSplit,
        uint16[5] calldata percentagesWETHSplit,
        uint16[5] calldata percentagesARBSplit,
        address recipient
    ) external whenNotPaused returns (uint256 usdcOut) {
        require(NSTin > 0, "dx=0");
        require(_getTotal(percentagesWBTCSplit) == 10000 && _getTotal(percentagesWETHSplit) == 10000 && _getTotal(percentagesARBSplit) == 10000, "!=100%");

        uint256 balanceWBtc = IERC20(tokens[1]).balanceOf(address(this));
        uint256 balanceWEth = IERC20(tokens[2]).balanceOf(address(this));
        uint256 balanceArb = IERC20(tokens[3]).balanceOf(address(this));
        uint256 wBtcIn = balanceWBtc * NSTin / totalSupply();
        uint256 wEthIn = balanceWEth * NSTin / totalSupply();
        uint256 ArbIn = balanceArb * NSTin / totalSupply();
        uint256[5] memory btcSplits;
        uint256[5] memory ethSplits;
        uint256[5] memory arbSplits;

        for (uint8 index = 0; index < 5;) {
            btcSplits[index] = wBtcIn * percentagesWBTCSplit[index] / 10000;
            ethSplits[index] = wEthIn * percentagesWETHSplit[index] / 10000;
            ethSplits[index] = ArbIn * percentagesWETHSplit[index] / 10000;
            unchecked {
                ++index;
            }
        }
        _burn(msg.sender, NSTin);

        approveAMM(1, wBtcIn, 5);
        approveAMM(2, wEthIn, 5);
        approveAMM(3, ArbIn, 5);
        TransferHelper.safeTransfer(
            tokens[0],
            recipient,
            usdcOut = swapWithParamsCustom(1, 0, btcSplits) + swapWithParamsCustom(2, 0, ethSplits) + swapWithParamsCustom(3, 0, arbSplits)
        );
        emit Burn(recipient, usdcOut, NSTin);
    }

    function _getTotal(uint16[5] memory _params) private pure returns (uint16) {
        uint256 len = _params.length;
        uint16 total = 0;
        for (uint8 i = 0; i < len;) {
            uint16 n = _params[i];
            if (n != 0) {
                total += n;
            }
            unchecked {
                ++i;
            }
        }
        return total;
    }

    //////////////////////////////////
    // SPECIAL PERMISSION FUNCTIONS//
    /////////////////////////////////

    function pause() external onlyOwner {
        _pause();
    }

    function unpause() external onlyOwner {
        _unpause();
    }

   
}
