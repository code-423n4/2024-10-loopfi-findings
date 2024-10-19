Q1. After a user calls ```initiateCooldown()```, he expects that the tokens will be available after the current ```cooldownPeriod```. Unfortunately, this is not true if ```cooldownPeriod``` is changed during this period by the owner to be a larger value. 

[https://github.com/code-423n4/2024-10-loopfi/blob/d219f0132005b00a68f505edc22b34f9a8b49766/src/Locking.sol#L59-L66](https://github.com/code-423n4/2024-10-loopfi/blob/d219f0132005b00a68f505edc22b34f9a8b49766/src/Locking.sol#L59-L66)

Scenario: User frank calls ```initiateCooldown(1 ether)``` with ```cooldownPeriod``` = 1 days. So he expects he can withdraw 1 ether of tokens after 1 day. However, during this time, the owner calls ```setCooldownPeriod()``` to change ```cooldownPeriod``` to 1 weeks. Such change should not have changed the existing CoolDown transactions. Now, frank can only withdraw his tokens after 1 weeks instead of 1 day.

Mitigation: introduce ```cooldownEnd``` and calculates it when ```initiateCooldown()``` is called. In this way, ```cooldownEnd``` will not be affected by later change of ```cooldownPeriod```. 
 
