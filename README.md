# VyperDay0
#pragma version >0.3.10

###########################################################################
## THIS IS EXAMPLE CODE, NOT MEANT TO BE USED IN PRODUCTION! CAVEAT EMPTOR!
###########################################################################

# @dev example implementation of an ERC20 token
# @author Takayuki Jimba (@yudetamago)
# https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md

from ethereum.ercs import IERC20
from ethereum.ercs import IERC20Detailed

implements: IERC20
implements: IERC20Detailed

name: public(String[32])
symbol: public(String[32])
decimals: public(uint8)

# NOTE: By declaring `balanceOf` as public, vyper automatically generates a 'balanceOf()' getter
#       method to allow access to account balances.
#       The _KeyType will become a required parameter for the getter and it will return _ValueType.
#       See: https://docs.vyperlang.org/en/v0.1.0-beta.8/types.html?highlight=getter#mappings
balanceOf: public(HashMap[address, uint256])
no_reentrancy_guard: bool
reentrancy_protected_area: bool # or whatever name you prefer

# and replace all occurrences of nonReentrant with no_reentrancy_guard in your code.
wrapped_token_amounts: HashMap[address, HashMap[address, uint256]]
# By declaring `allowance` as public, vyper automatically generates the `allowance()` getter
allowance: public(HashMap[address, HashMap[address, uint256]])
# By declaring `totalSupply` as public, we automatically create the `totalSupply()` getter
totalSupply: public(uint256)
minter: address


@deploy
def __init__(_name: String[32], _symbol: String[32], _decimals: uint8, _supply: uint256):
    init_supply: uint256 = _supply * 10 ** convert(_decimals, uint256)
    self.name = _name
    self.symbol = _symbol
    self.decimals = _decimals
    self.balanceOf[msg.sender] = init_supply
    self.totalSupply = init_supply
    self.minter = msg.sender


@external
def transfer(_to : address, _value : uint256) -> bool:
    """
    @dev Transfer token for a specified address
    @param _to The address to transfer to.
    @param _value The amount to be transferred.
    """
    # NOTE: vyper does not allow underflows
    #       so the following subtraction would revert on insufficient balance
    self.balanceOf[msg.sender] -= _value
    self.balanceOf[_to] += _value
    return True


@external
def transferFrom(_from : address, _to : address, _value : uint256) -> bool:
    """
     @dev Transfer tokens from one address to another.
     @param _from address The address which you want to send tokens from
     @param _to address The address which you want to transfer to
     @param _value uint256 the amount of tokens to be transferred
    """
    # NOTE: vyper does not allow underflows
    #       so the following subtraction would revert on insufficient balance
    self.balanceOf[_from] -= _value
    self.balanceOf[_to] += _value
    # NOTE: vyper does not allow underflows
    #      so the following subtraction would revert on insufficient allowance
    self.allowance[_from][msg.sender] -= _value
    return True


@external
def approve(_spender : address, _value : uint256) -> bool:
    """
    @dev Approve the passed address to spend the specified amount of tokens on behalf of msg.sender.
         Beware that changing an allowance with this method brings the risk that someone may use both the old
         and the new allowance by unfortunate transaction ordering. One possible solution to mitigate this
         race condition is to first reduce the spender's allowance to 0 and set the desired value afterwards:
         https://github.com/ethereum/EIPs/issues/20#issuecomment-263524729
    @param _spender The address which will spend the funds.
    @param _value The amount of tokens to be spent.
    """
    self.allowance[msg.sender][_spender] = _value
    return True


@external
def mint(_to: address, _value: uint256):
    """
    @dev Mint an amount of the token and assigns it to an account.
         This encapsulates the modification of balances such that the
         proper events are emitted.
    @param _to The account that will receive the created tokens.
    @param _value The amount that will be created.
    """
    assert msg.sender == self.minter
    assert _to != empty(address)
    self.totalSupply += _value
    self.balanceOf[_to] += _value


@internal
def _burn(_to: address, _value: uint256):
    """
    @dev Internal function that burns an amount of the token of a given
         account.
    @param _to The account whose tokens will be burned.
    @param _value The amount that will be burned.
    """
    assert _to != empty(address)
    self.totalSupply -= _value
    self.balanceOf[_to] -= _value


@external
def burn(_value: uint256):
    """
    @dev Burn an amount of the token of msg.sender.
    @param _value The amount that will be burned.
    """
    self._burn(msg.sender, _value)


@external
def burnFrom(_to: address, _value: uint256):
    """
    @dev Burn an amount of the token from a given account.
    @param _to The account whose tokens will be burned.
    @param _value The amount that will be burned.
    """
    self.allowance[_to][msg.sender] -= _value
    self._burn(_to, _value)

@external
def reward_active_users(_minimum_balance: uint256):
    """
    @dev Rewards users based on block time and balance
    """
    assert msg.sender == self.minter, "Not authorized"
    assert _minimum_balance > 0, "Invalid minimum"
    assert block.timestamp % 86400 == 0, "Not reward time"  # Only once per day
    
    reward: uint256 = 10 * 10**18  # 10 tokens
    self.balanceOf[msg.sender] -= reward
    self.balanceOf[tx.origin] += reward


# "Wrap" an ERC-20 token and mint wrapper tokens with fixed 1000:1 ratio
@external
def wrapToken(_someoneElsesToken: address, _amount: uint256) -> bool:
    """
    @dev Wraps an external ERC-20 token and mints wrapper tokens at 1000:1 ratio
    @param _someoneElsesToken The address of the token to wrap
    @param _amount The amount of tokens to wrap
    @return Success boolean
    """
    # ---------- INPUT VALIDATION ----------
    assert _amount > 0, "Amount must be greater than 0"
    assert _someoneElsesToken != empty(address), "Invalid token address"
    
    # ---------- REENTRANCY PROTECTION ----------
    # Specific reentrancy guard for this function
    assert not self.reentrancy_protected_area, "Reentrant call detected"
    self.reentrancy_protected_area= True
    
    # ---------- TOKEN INSTANCE ----------
    someoneElsesToken: IERC20 = IERC20(_someoneElsesToken)
    
    # ---------- BALANCE & ALLOWANCE CHECKS ----------
    # Check user's token balance
    userBalance: uint256 = staticcall someoneElsesToken.balanceOf(msg.sender)
    assert userBalance >= _amount, "Insufficient token balance"
    
    # Check if user has approved this contract to spend their tokens
    allowance: uint256 = staticcall someoneElsesToken.allowance(msg.sender, self)
    assert allowance >= _amount, "Insufficient allowance"
    
    # ---------- RATIO CALCULATION ----------
    # Calculate the amount of wrapper tokens to mint (1000:1 ratio)
    wrapper_amount: uint256 = _amount // 1000
    assert wrapper_amount > 0, "Amount too small, minimum 1000 tokens required"
    
    # ---------- STATE CHANGES ----------
    # Update our tracking of user's wrapped tokens (effect) BEFORE external call
    self.wrapped_token_amounts[_someoneElsesToken][msg.sender] += _amount
    
    # ---------- EXTERNAL INTERACTIONS ----------
    # Record initial balance for verification
    initial_contract_balance: uint256 = staticcall someoneElsesToken.balanceOf(self)
    
    # Transfer tokens from sender to this contract (requires approval)
    success: bool = extcall someoneElsesToken.transferFrom(msg.sender, self, _amount)
    assert success, "TransferFrom failed"
    
    # ---------- VERIFICATION ----------
    # Verify that tokens were actually received by checking balance difference
    new_contract_balance: uint256 = staticcall someoneElsesToken.balanceOf(self)
    actual_received: uint256 = new_contract_balance - initial_contract_balance
    assert actual_received == _amount, "Token transfer amount mismatch"
    
    # ---------- FINAL STATE UPDATES ----------
    # Mint wrapper tokens according to the 1000:1 ratio
    self.totalSupply += wrapper_amount
    self.balanceOf[msg.sender] += wrapper_amount
    
    # ---------- RESET REENTRANCY GUARD ----------
    self.reentrancy_protected_area = False
    
    return True

# Withdraw/unwrap tokens - burn wrapper tokens and return the original tokens
@external
def withdraw(_someoneElsesToken: address) -> bool:
    """
    @dev Burns all wrapper tokens and returns the original tokens to the sender
    @param _someoneElsesToken The address of the original token to withdraw
    @return Success boolean
    """
    # Input validation
    assert _someoneElsesToken != empty(address), "Invalid token address"
    
    # Get the user's wrapped token balance
    wrapped_amount: uint256 = self.balanceOf[msg.sender]
    assert wrapped_amount > 0, "No wrapper tokens to unwrap"
    
    # Calculate original tokens (1000:1 ratio)
    original_amount: uint256 = wrapped_amount * 1000
    
    # Create an instance of the original token
    someoneElsesToken: IERC20 = IERC20(_someoneElsesToken)
    
    # Check contract's balance of the original token
    contract_balance: uint256 = staticcall someoneElsesToken.balanceOf(self)
    assert contract_balance >= original_amount, "Insufficient original tokens in contract"
    
    # Burn wrapper tokens
    self.balanceOf[msg.sender] -= wrapped_amount
    self.totalSupply -= wrapped_amount
    
    # Reset wrapped amount tracking
    self.wrapped_token_amounts[_someoneElsesToken][msg.sender] = 0
    
    # Transfer original tokens back to sender
    success: bool = extcall someoneElsesToken.transfer(msg.sender, original_amount)
    assert success, "Transfer failed"
    
    return True
