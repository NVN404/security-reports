# Audit Arena Week 4 Findings (From CSV)

Source: `NVN404_Submissions.csv`

## 1. [C1]-Uninitialized Storage Pointer in updateConfig Overwrites saleToken and admin storage slots

- Date: 2026-02-24T03:01:12.684146
- Severity: Critical
- Scope: PresaleEngine.sol:updateConfig()
- Reporter: yun0hu

### Details

[C1]-Uninitialized Storage Pointer in updateConfig Overwrites saleToken and admin storage slots 

The updateConfig function declares a local PresaleConfig storage c pointer without initializing it to any state variable in the contract .
 This causes it to default to storage slot 0. Writing to its fields silently overwrites saleToken (slot 0) and admin (slot 1), permanently corrupting the contract's two most critical state variables .
Once an admin calls updateConfig(_min , _max ), all ETH sent via buyTokens() becomes permanently locked (the saleToken address is destroyed), and admin access is lost (the admin address is overwritten). This is a total loss of all contract funds and control.

root cause :
There is no PresaleConfig state variable declared in the contract. The local storage pointer has nowhere to point, so Solidity defaults it to slot 0. The contract's storage layout is:
Slot 0: saleToken
Slot 1: admin
Slot 2: price

attack path :
admin calls updateconfig with min and max values 
As the storage is pointed to sale token and admin storage slots , the data is overwritten
making the contract lose its functions and locking of funds ,
admin becomes helpless as the key is destroyed and tokens are sent to a dead adress .

recommended fix:
  mapping(address => uint256) public contributions;
+ PresaleConfig public config;  // add state variable

  constructor(address _token) {
      saleToken = IERC20(_token);
      admin = msg.sender;
  }

  function updateConfig(uint256 _min, uint256 _max) external {
      require(msg.sender == admin, "Auth");

-     PresaleConfig storage c;
+     PresaleConfig storage c = config; // point to the variable declared to avoid storage collision

      c.minContribution = _min;
      c.maxContribution = _max;
  }

---

## 2. [M1] - buyTokens() Accepts ETH and Delivers Zero Tokens When msg.value < price

- Date: 2026-02-25T10:00:16.948083
- Severity: Medium
- Scope: presaleEngine.sol:buyTokens()
- Reporter: yun0hu

### Details

[M1] - buyTokens() Accepts ETH and Delivers Zero Tokens When msg.value < price

When a user calls buyTokens() with msg.value less than price (100 wei), integer division truncates amount to exactly 0. The transaction still succeeds with the users ETH is accepted, contributions is incremented, and a 0 token transfer completes. The user pays ETH, receives nothing, and the ETH is permanently locked in the contract having no withdrawal mechanism .

attack path :
User calls buyTokens() with value 99wei
amount = (99 / 100) * 10**18 = 0 * 10**18 = 0.
contributions[msg.sender] += 99  =>  ETH is recorded as contributed.
saleToken.transfer(msg.sender, 0) => transfers 0 tokens, returns true.
Transaction succeeds. User paid 99 wei, received 0 tokens.
ETH is permanently locked with no withdraw function, no refund path.

recommended fix :
add checks before transfering and updating contributions 
+   require(amount > 0, "Contribution too small");

---
