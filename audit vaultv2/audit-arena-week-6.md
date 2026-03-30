# Audit Arena Week 6 Findings (From CSV)

Source: `NVN404_Submissions.csv`

## 1. Missing access control on updateRate()

- Date: 2026-03-09T09:16:01.144785
- Severity: High
- Scope: audit-arena-week-6.sol:updateRate()
- Reporter: yun0hu

### Details

Missing access control on updateRate()

Anyone can call updateRate() and set rewardRatePerSecond to any value, including 0 or an very high number 

root cause : no access specifiers

attack path : 
An attacker sets up an account with a valid lastUpdate timestamp.
the attacker calls updateRate(type(uint256).max) and sets it to high rate .
the attacker immediately calls claimRewards(). The contract computes their accrued time and multiplies it by the massively high rate.
the contract transfers the maximum possible balance of the reward tokens to the attacker.

recommended fix : 
function updateRate(uint256 newRate) external {
    require(msg.sender == owner, "Not owner");
    rewardRatePerSecond = newRate;
}

---

## 2. Reward calculation is independent of user's staked balance, allowing anyone to drain the reward pool for free

- Date: 2026-03-09T09:33:02.555688
- Severity: High
- Scope: audit-arena-week-6.sol:_claimRewards()
- Reporter: yun0hu

### Details

Reward calculation is independent of user's staked balance, allowing anyone to drain the reward pool for free

the _claimRewards function calculates a user's reward simply by multiplying the elapsed time. any address can interact with the contract to start a timer, wait, and claim rewards proportionate to the time elapsed , even if they have zero tokens staked. this allows attackers to easily drain the entire reward pool without risking or locking any of thier capital.

root cause :in _claimRewards, the calculation uint256 reward = elapsed * rewardRatePerSecond; fails to account for the user's share of the pool (staked[user]).

attack path :
An attacker calls claimRewards() with 0 tokens staked. This bypasses the checks and initializes their lastUpdate[msg.sender] to the current block.timestamp.
The attacker waits for a certain amount of time T.
The attacker calls claimRewards() again.
The contract computes the reward as elapsed * rewardRatePerSecond.
The contract transfers the calculated reward amount to the attacker, successfully draining funds despite the attacker staking 0 tokens.

recommended fix : 

we should mulitply the reward with staked[user]
uint256 reward = (elapsed * staked[user] * rewardRatePerSecond) ;

this gives the reward according to the weight of the capital staked by the user

---

## 3. Incorrect balance check in stakeTokenBalance returns ETH balance instead of Token balance

- Date: 2026-03-09T09:38:31.289802
- Severity: High
- Scope: audit-arena-week-6.sol:stakeTokenBalance()
- Reporter: yun0hu

### Details

Incorrect balance check in stakeTokenBalance returns ETH balance instead of Token balance

The stakeTokenBalance function returns address(stakeToken).balance, which gives the balance of ETH balance of the token contract itself instead of balance of the ERC20 token balance . 

root cause : the code uses .balance on the token address instead of calling balanceOf(address(this)).

attack path :
when the owner attempts to use emergencyWithdraw() to recover mistakenly sent tokens or to migrate the pool.
The function calculates the amount to transfer using stakeTokenBalance().
Because address(stakeToken).balance is 0, the contract transfers 0 tokens to the owner, failing to perform the withdrawal.
leading to not able to transfer tokens to safe place in case of emergency.

recommendation :
function stakeTokenBalance() public view returns (uint256) {
    return stakeToken.balanceOf(address(this));
}

---

## 4. Use of tx.origin for authorization makes the contract vulnerable to phishing

- Date: 2026-03-09T09:50:26.414023
- Severity: Medium
- Scope: -
- Reporter: @yun0hu

### Details

Use of tx.origin for authorization makes the contract vulnerable to phishing

The emergencyWithdraw function uses require(tx.origin == owner) instead of msg.sender == owner. This allows a malicious contract to "trick" the owner into calling it, which then calls SimpleStaking and successfully passes the check, allowing the attacker to trigger a withdrawal of all funds from the contract to the owner's address (disrupting the protocol).
(it requires phishing the owner with fake malicious contract )

root cause : usage of tx.origin 

attack path :
Attacker creates a malicious contract that calls SimpleStaking.emergencyWithdraw().
attacker tricks the owner into calling a function on the malicious contract.
the malicious contract calls SimpleStaking, and since the transaction was started by the owner, tx.origin == owner evaluates to true.

recommendation :
require(msg.sender == owner, "Not owner");
always use msg.sender to avoid the above issues

---

## 5. Rewards are paid out of user principal due to unsegregated balances, leading to loss of staked deposits

- Date: 2026-03-09T11:11:27.365339
- Severity: High
- Scope: -
- Reporter: @yun0hu

### Details

Rewards are paid out of user principal due to unsegregated balances, leading to loss of staked deposits

The contract holds both the staked tokens and the reward tokens in the same balance (stakeToken.balanceOf(address(this))), but it fails to maintain separate accounting to protect user deposits. When _claimRewards is executed, it transfers tokens directly from the contract's overall balance. If the contract has not been sufficiently funded with rewards by the owner, it will seamlessly use the deposited stakes of other users to pay out the yield. This leads to a direct loss of user principal

root cause :
lack of accounting separation.

attack path :
alice and Bob each stake 1000 tokens (Contract balance = 2000).
 owner never or forgets to call fundRewards().
time passes, and Alice has accrued 500 tokens in rewards.
Alice calls claimRewards(). The contract sends her 500 tokens from its 2000 token balance.
The contract balance is now 1500 tokens, but Alice and Bob still have 1000 tokens each recorded in their staked mapping.
500 tokens of Bob's principal have essentially been stolen to pay Alice's reward. 
Bob will inevitably be unable to withdraw his full stake later.


even if the owner funds the contract through fundRewards()
the reward and users principal are both stored in the same contract and not segregated .
with more number of user , the total amount will be mixed up with both rewards and staked deposits .
so either we should track the total amount of staked tokens or the rewards to avoid this issue 
totalStaked += amount;
or 
rewardPool += amount;

---

## 6. Incorrect reward calculation and update destroys previously accrued user yield

- Date: 2026-03-09T11:52:53.127422
- Severity: High
- Scope: -
- Reporter: @yun0hu

### Details

Incorrect reward calculation and update destroys previously accrued user yield

When updateRate() modifies rewardRatePerSecond, the new rate is applied to the entire timespan i.e, a user's lastUpdate timestamp. Because the contract uses elapsed * currentRate instead of caching rewards before a rate change, any adjustment to the rate alters the yield pf the users who already earned in the past. If the rate is decreased, users suffer a direct, mathematical loss of  previously accrued funds.

root cause :
 uint256 reward = elapsed * rewardRatePerSecond; mathematically assumes the rate has been constant since lastUpdate. It fails to account for rate fluctuations during the elapsed window.

attack path :

users stake tokens for 6 months at rewardRatePerSecond = 10. They have already earned 6 months' worth of yield.
The owner updates the rate to 5 by calling updateRate(5).
due to the incorrect calculation in _claimRewards, the contract calculates the past 6 months using the new rate of 5.
all users immediately lose exactly half of the rewards they had already earned prior to the update.

recommended path :
we should take snapshot of each user before updating to new rate 
or we should use different variable to track the old rate and new rate

---

## 7. State variables are updated after external calls, opening the contract to Reentrancy attack

- Date: 2026-03-11T04:16:17.388598
- Severity: High
- Scope: audit-arena-week-6.sol:_claimRewards()
- Reporter: @yun0hu

### Details

State variables are updated after external calls, opening the contract to Reentrancy attack

In _claimRewards, the stakeToken.transfer call is made before the user's lastUpdate state is updated. This severely violates the Checks-Effects-Interactions (CEI) pattern. If the ERC20 token allows for callbacks (such as ERC777 tokens or tokens with hook implementations), an attacker can hijack the execution flow during the transfer and re-enter the contract's claimRewards() or withdraw() function

root cause :
the lastUpdate[user] = block.timestamp; state interaction occurs after the external call stakeToken.transfer(user, reward);

attack path :
The staking protocol uses a token that supports backward compatible receiver hooks.
An attacker stakes and waits for a reward to accrue.
attacker calls claimRewards().
_claimRewards calculates the reward and calls stakeToken.transfer(attacker, reward).
The token contract executes the transfer and triggers a callback function on the attacker's smart contract.
Inside the hook, the attacker's contract calls claimRewards() again.
Since lastUpdate was not yet updated by the previous execution, elapsed is still the same, and the contract initiates another transfer of the same reward.
The loop continues until the contract's balance is drained.


recommended fix :
function _claimRewards(address user) internal {
// check
    if (lastUpdate[user] == 0) {
        lastUpdate[user] = block.timestamp;
        return;
    }
    uint256 elapsed = block.timestamp - lastUpdate[user];
    uint256 reward = elapsed * rewardRatePerSecond;

    // EFFECTS
    lastUpdate[user] = block.timestamp;

    // INTERACTIONS
    if (reward > 0) {
        stakeToken.transfer(user, reward);
    }
}

---
