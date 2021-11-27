# CACA

This repository was created by Official BlockAudit Team.

<a href="https://sm.ms/image/rXptNGQ1aubMKsS" target="_blank"><img src="https://i.loli.net/2021/11/27/rXptNGQ1aubMKsS.jpg" /></a>


// SPDX-License-Identifier: MIT
pragma solidity ^0.6.12;

import "./CACA.sol";

contract CACA is Context, IERC20, Ownable {
    using SafeMath for uint256;
    using Address for address;


    struct RValuesStruct {
        uint256 rAmount;
        uint256 rTransferAmount;
        uint256 rFee;
        uint256 rBurn;
        uint256 rCharity;
        uint256 rLiquidity;
    }

    struct TValuesStruct {
        uint256 tTransferAmount;
        uint256 tFee;
        uint256 tBurn;
        uint256 tCharity;
        uint256 tLiquidity;
    }

    struct ValuesStruct {
        uint256 rAmount;
        uint256 rTransferAmount;
        uint256 rFee;
        uint256 rBurn;
        uint256 rCharity;
        uint256 rLiquidity;
        uint256 tTransferAmount;
        uint256 tFee;
        uint256 tBurn;
        uint256 tCharity;
        uint256 tLiquidity;
    }

    mapping (address => uint256) private _rOwned;
    mapping (address => uint256) private _tOwned;
    mapping (address => mapping (address => uint256)) private _allowances;

    mapping (address => bool) private _isExcludedFromFee;

    mapping (address => bool) private _isExcluded;
    address[] private _excluded;

    uint256 private constant MAX = ~uint256(0);
    uint256 private _tTotal = 100000000 * 10**9;
    uint256 private _rTotal = (MAX - (MAX % _tTotal));
    uint256 private _tFeeTotal;
    uint256 private _tBurnTotal;

    string private _name = "CACA";
    string private _symbol = "CACA";
    uint8 private _decimals = 9;

    uint256 public _taxFee = 0;
    uint256 private _previousTaxFee = _taxFee;

    uint256 public _burnFee = 0;
    uint256 private _previousBurnFee = _burnFee;

    uint256 public _charityFee = 0;
    uint256 private _previousCharityFee = _charityFee;

    uint256 public _liquidityFee = 0;
    uint256 private _previousLiquidityFee = _liquidityFee;

    address public charityWallet = 0x69cae5aB33F7d3e4d9c00F7A14AbdF2a20f784e1;

    IUniswapV2Router02 public immutable uniswapV2Router;
    address public immutable uniswapV2Pair;

    bool inSwapAndLiquify;
    bool public swapAndLiquifyEnabled = false;

    uint256 public _maxTxAmount = 100000000 * 10**9;
    uint256 private numTokensSellToAddToLiquidity = 500000 * 10**9;

    event MinTokensBeforeSwapUpdated(uint256 minTokensBeforeSwap);
    event SwapAndLiquifyEnabledUpdated(bool enabled);
    event SwapAndLiquify(
        uint256 tokensSwapped,
        uint256 ethReceived,
        uint256 tokensIntoLiqudity
    );

    modifier lockTheSwap {
        inSwapAndLiquify = true;
        _;
        inSwapAndLiquify = false;
    }

    constructor () public {
        _rOwned[_msgSender()] = _rTotal;

        IUniswapV2Router02 _uniswapV2Router = IUniswapV2Router02(0x10ED43C718714eb63d5aA57B78B54704E256024E);
        uniswapV2Pair = IUniswapV2Factory(_uniswapV2Router.factory())
        .createPair(address(this), _uniswapV2Router.WETH());
        uniswapV2Router = _uniswapV2Router;
        _isExcludedFromFee[owner()] = true;
        _isExcludedFromFee[address(this)] = true;

        emit Transfer(address(0), _msgSender(), _tTotal);
    }

    function name() public view returns (string memory) {
        return _name;
    }

    function symbol() public view returns (string memory) {
        return _symbol;
    }

    function decimals() public view returns (uint8) {
        return _decimals;
    }

    function totalSupply() public view override returns (uint256) {
        return _tTotal;
    }

    function balanceOf(address account) public view override returns (uint256) {
        if (_isExcluded[account]) return _tOwned[account];
        return tokenFromReflection(_rOwned[account]);
    }

    function transfer(address recipient, uint256 amount) public override returns (bool) {
        _transfer(_msgSender(), recipient, amount);
        return true;
    }

    function allowance(address owner, address spender) public view override returns (uint256) {
        return _allowances[owner][spender];
    }

    function approve(address spender, uint256 amount) public override returns (bool) {
        _approve(_msgSender(), spender, amount);
        return true;
    }

    function transferFrom(address sender, address recipient, uint256 amount) public override returns (bool) {
        _transfer(sender, recipient, amount);
        _approve(sender, _msgSender(), _allowances[sender][_msgSender()].sub(amount, "ERC20: transfer amount exceeds allowance"));
        return true;
    }

    function increaseAllowance(address spender, uint256 addedValue) public virtual returns (bool) {
        _approve(_msgSender(), spender, _allowances[_msgSender()][spender].add(addedValue));
        return true;
    }

    function decreaseAllowance(address spender, uint256 subtractedValue) public virtual returns (bool) {
        _approve(_msgSender(), spender, _allowances[_msgSender()][spender].sub(subtractedValue, "ERC20: decreased allowance below zero"));
        return true;
    }

    function isExcludedFromReward(address account) public view returns (bool) {
        return _isExcluded[account];
    }

    function totalFees() public view returns (uint256) {
        return _tFeeTotal;
    }

    function totalBurn() public view returns (uint256) {
        return _tBurnTotal;
    }

    function deliver(uint256 tAmount) public {
        address sender = _msgSender();
        require(!_isExcluded[sender], "Excluded addresses cannot call this function");
        uint256 rAmount = _getValues(tAmount).rAmount;
        _rOwned[sender] = _rOwned[sender].sub(rAmount);
        _rTotal = _rTotal.sub(rAmount);
        _tFeeTotal = _tFeeTotal.add(tAmount);
    }

    function reflectionFromToken(uint256 tAmount, bool deductTransferFee) public view returns(uint256) {
        require(tAmount <= _tTotal, "Amount must be less than supply");
        if (!deductTransferFee) {
            uint256 rAmount = _getValues(tAmount).rAmount;
            return rAmount;
        } else {
            uint256 rTransferAmount = _getValues(tAmount).rTransferAmount;
            return rTransferAmount;
        }
    }

    function tokenFromReflection(uint256 rAmount) public view returns(uint256) {
        require(rAmount <= _rTotal, "Amount must be less than total reflections");
        uint256 currentRate =  _getRate();
        return rAmount.div(currentRate);
    }

    function excludeFromReward(address account) public onlyOwner() {
        require(account != 0x10ED43C718714eb63d5aA57B78B54704E256024E, 'We can not exclude Pancake router.');
        require(!_isExcluded[account], "Account is already excluded");
        require(_excluded.length < 50, "Excluded list is too long");
        if(_rOwned[account] > 0) {
            _tOwned[account] = tokenFromReflection(_rOwned[account]);
        }
        _isExcluded[account] = true;
        _excluded.push(account);
    }

    function includeInReward(address account) external onlyOwner() {
        require(_isExcluded[account], "Account is already excluded");
        for (uint256 i = 0; i < _excluded.length; i++) {
            if (_excluded[i] == account) {
                _excluded[i] = _excluded[_excluded.length - 1];
                _tOwned[account] = 0;
                _isExcluded[account] = false;
                _excluded.pop();
                break;
            }
        }
    }
    
    receive() external payable {}

    function _distributeFee(uint256 rFee, uint256 rBurn, uint256 rCharity, uint256 tFee, uint256 tBurn, uint256 tCharity) private {
        _rTotal = _rTotal.sub(rFee).sub(rBurn);
        _tFeeTotal = _tFeeTotal.add(tFee);
        _tTotal = _tTotal.sub(tBurn);
        _tBurnTotal = _tBurnTotal.add(tBurn);

        _rOwned[charityWallet] = _rOwned[charityWallet].add(rCharity);
        if (_isExcluded[charityWallet]) {
            _tOwned[charityWallet] = _tOwned[charityWallet].add(tCharity);
        }
    }

    function _getValues(uint256 tAmount) private view returns (ValuesStruct memory) {
        TValuesStruct memory tvs = _getTValues(tAmount);
        RValuesStruct memory rvs = _getRValues(tAmount, tvs.tFee, tvs.tBurn, tvs.tCharity, tvs.tLiquidity, _getRate());

        return ValuesStruct(
            rvs.rAmount,
            rvs.rTransferAmount,
            rvs.rFee,
            rvs.rBurn,
            rvs.rCharity,
            rvs.rLiquidity,
            tvs.tTransferAmount,
            tvs.tFee,
            tvs.tBurn,
            tvs.tCharity,
            tvs.tLiquidity
        );
    }

    function _getTValues(uint256 tAmount) private view returns (TValuesStruct memory) {
        uint256 tFee = calculateTaxFee(tAmount);
        uint256 tBurn = calculateBurnFee(tAmount);
        uint256 tCharity = calculateCharityFee(tAmount);
        uint256 tLiquidity = calculateLiquidityFee(tAmount);
        uint256 tTransferAmount = tAmount.sub(tFee).sub(tBurn).sub(tCharity).sub(tLiquidity);
        return TValuesStruct(tTransferAmount, tFee, tBurn, tCharity, tLiquidity);
    }

    function _getRValues(uint256 tAmount, uint256 tFee, uint256 tBurn, uint256 tCharity, uint256 tLiquidity, uint256 currentRate) private pure returns (RValuesStruct memory) {
        uint256 rAmount = tAmount.mul(currentRate);
        uint256 rFee = tFee.mul(currentRate);
        uint256 rBurn = tBurn.mul(currentRate);
        uint256 rCharity = tCharity.mul(currentRate);
        uint256 rLiquidity = tLiquidity.mul(currentRate);
        uint256 rTransferAmount = rAmount.sub(rFee).sub(rLiquidity).sub(rBurn).sub(rCharity);
        return RValuesStruct(rAmount, rTransferAmount, rFee, rBurn, rCharity, rLiquidity);
    }

    function _getRate() private view returns(uint256) {
        (uint256 rSupply, uint256 tSupply) = _getCurrentSupply();
        return rSupply.div(tSupply);
    }

    function _getCurrentSupply() private view returns(uint256, uint256) {
        uint256 rSupply = _rTotal;
        uint256 tSupply = _tTotal;
        for (uint256 i = 0; i < _excluded.length; i++) {
            if (_rOwned[_excluded[i]] > rSupply || _tOwned[_excluded[i]] > tSupply) return (_rTotal, _tTotal);
            rSupply = rSupply.sub(_rOwned[_excluded[i]]);
            tSupply = tSupply.sub(_tOwned[_excluded[i]]);
        }
        if (rSupply < _rTotal.div(_tTotal)) return (_rTotal, _tTotal);
        return (rSupply, tSupply);
    }

    function _takeLiquidity(uint256 rLiquidity, uint256 tLiquidity) private {
        _rOwned[address(this)] = _rOwned[address(this)].add(rLiquidity);
        if(_isExcluded[address(this)])
            _tOwned[address(this)] = _tOwned[address(this)].add(tLiquidity);
    }

    function calculateTaxFee(uint256 _amount) private view returns (uint256) {
        return _amount.mul(_taxFee).div(
            10**2
        );
    }

    function calculateBurnFee(uint256 _amount) private view returns (uint256) {
        return _amount.mul(_burnFee).div(
            10**2
        );
    }

    function calculateCharityFee(uint256 _amount) private view returns (uint256) {
        return _amount.mul(_charityFee).div(
            10**2
        );
    }

    function calculateLiquidityFee(uint256 _amount) private view returns (uint256) {
        return _amount.mul(_liquidityFee).div(
            10**2
        );
    }

    function removeAllFee() private {
        _taxFee       = 0;
        _liquidityFee = 0;
        _burnFee      = 0;
        _charityFee = 0;
    }

    function restoreAllFee() private {
        _taxFee       = 0;
        _liquidityFee = 2;
        _charityFee = 2;
        _burnFee      = 2;

    }

    function isExcludedFromFee(address account) public view returns(bool) {
        return _isExcludedFromFee[account];
    }

    function _approve(address owner, address spender, uint256 amount) private {
        require(owner != address(0), "ERC20: approve from the zero address");
        require(spender != address(0), "ERC20: approve to the zero address");

        _allowances[owner][spender] = amount;
        emit Approval(owner, spender, amount);
    }

    function _transfer(
        address from,
        address to,
        uint256 amount
    ) private {
        require(from != address(0), "ERC20: transfer from the zero address");
        require(to != address(0), "ERC20: transfer to the zero address");
        require(amount > 0, "Transfer amount must be greater than zero");

        uint256 contractTokenBalance = balanceOf(address(this));
        bool overMinTokenBalance = contractTokenBalance >= numTokensSellToAddToLiquidity;
        if (
            overMinTokenBalance &&
            !inSwapAndLiquify &&
            from != uniswapV2Pair &&
            swapAndLiquifyEnabled
        ) {
            contractTokenBalance = numTokensSellToAddToLiquidity;
            swapAndLiquify(contractTokenBalance);
        }

        _tokenTransfer(from,to,amount);
    }

    function swapAndLiquify(uint256 contractTokenBalance) private lockTheSwap {
        uint256 half = contractTokenBalance.div(2);
        uint256 otherHalf = contractTokenBalance.sub(half);
        uint256 initialBalance = address(this).balance;
        swapTokensForEth(half);
        uint256 newBalance = address(this).balance.sub(initialBalance);
        addLiquidity(otherHalf, newBalance);

        emit SwapAndLiquify(half, newBalance, otherHalf);
    }

    function swapTokensForEth(uint256 tokenAmount) private {
        address[] memory path = new address[](2);
        path[0] = address(this);
        path[1] = uniswapV2Router.WETH();

        _approve(address(this), address(uniswapV2Router), tokenAmount);
        uniswapV2Router.swapExactTokensForETHSupportingFeeOnTransferTokens(
            tokenAmount,
            0,
            path,
            address(this),
            block.timestamp
        );
    }

    function addLiquidity(uint256 tokenAmount, uint256 ethAmount) private {
        _approve(address(this), address(uniswapV2Router), tokenAmount);
        uniswapV2Router.addLiquidityETH{value: ethAmount}(
            address(this),
            tokenAmount,
            0,
            0,
            owner(),
            block.timestamp
        );
    }

    function _tokenTransfer(address sender, address recipient, uint256 amount) private {
        if(_isExcludedFromFee[sender] || _isExcludedFromFee[recipient]){
            removeAllFee();
        }
        else{
            require(amount <= _maxTxAmount, "Transfer amount exceeds the maxTxAmount.");
        }

        ValuesStruct memory vs = _getValues(amount);
        _takeLiquidity(vs.rLiquidity, vs.tLiquidity);
        _distributeFee(vs.rFee, vs.rBurn, vs.rCharity, vs.tFee, vs.tBurn, vs.tCharity);

        if (_isExcluded[sender] && !_isExcluded[recipient]) {
            _transferFromExcluded(sender, recipient, amount, vs);
        } else if (!_isExcluded[sender] && _isExcluded[recipient]) {
            _transferToExcluded(sender, recipient, vs);
        } else if (!_isExcluded[sender] && !_isExcluded[recipient]) {
            _transferStandard(sender, recipient, vs);
        } else if (_isExcluded[sender] && _isExcluded[recipient]) {
            _transferBothExcluded(sender, recipient, amount, vs);
        }

        if(_isExcludedFromFee[sender] || _isExcludedFromFee[recipient])
            restoreAllFee();
    }

    function _transferStandard(address sender, address recipient, ValuesStruct memory vs) private {
        _rOwned[sender] = _rOwned[sender].sub(vs.rAmount);
        _rOwned[recipient] = _rOwned[recipient].add(vs.rTransferAmount);
        emit Transfer(sender, recipient, vs.tTransferAmount);
    }

    function _transferToExcluded(address sender, address recipient, ValuesStruct memory vs) private {
        _rOwned[sender] = _rOwned[sender].sub(vs.rAmount);
        _tOwned[recipient] = _tOwned[recipient].add(vs.tTransferAmount);
        _rOwned[recipient] = _rOwned[recipient].add(vs.rTransferAmount);
        emit Transfer(sender, recipient, vs.tTransferAmount);
    }

    function _transferFromExcluded(address sender, address recipient, uint256 tAmount, ValuesStruct memory vs) private {
        _tOwned[sender] = _tOwned[sender].sub(tAmount);
        _rOwned[sender] = _rOwned[sender].sub(vs.rAmount);
        _rOwned[recipient] = _rOwned[recipient].add(vs.rTransferAmount);
        emit Transfer(sender, recipient, vs.tTransferAmount);
    }

    function _transferBothExcluded(address sender, address recipient, uint256 tAmount, ValuesStruct memory vs) private {
        _tOwned[sender] = _tOwned[sender].sub(tAmount);
        _rOwned[sender] = _rOwned[sender].sub(vs.rAmount);
        _tOwned[recipient] = _tOwned[recipient].add(vs.tTransferAmount);
        _rOwned[recipient] = _rOwned[recipient].add(vs.rTransferAmount);
        emit Transfer(sender, recipient, vs.tTransferAmount);
    }

    function excludeFromFee(address account) public onlyOwner {
        _isExcludedFromFee[account] = true;
    }

    function includeInFee(address account) public onlyOwner {
        _isExcludedFromFee[account] = false;
    }

    function enableAllFees() external onlyOwner() {
        _taxFee       = 0;
        _previousTaxFee = _taxFee;
        _burnFee      = 2;
        _previousBurnFee = _taxFee;
        _charityFee = 2;
        _previousCharityFee = _charityFee;
        _liquidityFee = 2;
        _previousLiquidityFee = _liquidityFee;
        setSwapAndLiquifyEnabled(true);
    }

    function ExcludeAllFees() external onlyOwner() {
        _taxFee       = 0;
        _previousTaxFee = _taxFee;
        _burnFee      = 0;
        _previousBurnFee = _taxFee;
        _charityFee = 0;
        _previousCharityFee = _charityFee;
        _liquidityFee = 0;
        _previousLiquidityFee = _liquidityFee;
        setSwapAndLiquifyEnabled(false);
    }

    function setNewCharityWallet(address newWallet) external onlyOwner() {
        charityWallet = newWallet;
    }

    function setMaxTxPercent(uint256 maxTxPercent) external onlyOwner() {
        _maxTxAmount = _tTotal.mul(maxTxPercent).div(
            10**2
        );
    }

    function setSwapAndLiquifyEnabled(bool _enabled) public onlyOwner {
        swapAndLiquifyEnabled = _enabled;
        emit SwapAndLiquifyEnabledUpdated(_enabled);
    }
