# FIAT DAO Governance Migration

### Overview

FIATDAO intends to switch out their current governance system for the system developed by Element Finance. This document outlines a migration path.

Specifically, we will focus on the following questions:
- What are our options (both technical or economical) for extricating locked FDT?
- What would the deployment process for Element contracts look like? 
- What lessons can be learned from Element's own roll-out? (e.g. Elf NFTs, zk airdrop, GSC delegation)
- What is the feasibility of deploying a v2.1 of the DAO with the expectation of a v2.2 shortly thereafter? (i.e. we make the move initially without the curve gauge functionality, only to then later update it with new gov vaults once gauge functionality is live)

### Technical overview of Element's System

As mentioned in [this document](https://github.com/charlesndalton/governance-contract-comparison), the Element system comprises of a central *[CoreVoting](https://github.com/element-fi/council/blob/main/contracts/CoreVoting.sol)* contract, similar in design to Compound's GovernorBravo, with vote counting done by a series of authorized 'Voting Vaults.' All voting vaults must implement the interface *[IVotingVault](https://github.com/element-fi/council/blob/main/contracts/interfaces/IVotingVault.sol)*, which means that they need to implement the *queryVotePower* function. 

When a user votes on a governance proposal, they call the *[vote](https://github.com/element-fi/council/blob/0582395e6d720ae4b907bfda4ecd288e80794052/contracts/CoreVoting.sol#L211)* function of the *CoreVoting* contract, specifying all of the voting vaults that give them voting power. For example, if Alice is a team member with vesting tokens and has locked some of her vested tokens for voting power, she would pass in the addresses of the *VestingVault* and *LockingVault* contracts. *CoreVoting* would query her voting power on both of these vaults, add these values, and apply her total voting power to her proposal of choice.

It's important to note that Element's *LockingVault* is *not* like FIAT's vFDT, where a user can lock their FDT for additional voting power. It's more analogous to staking, where depositing tokens in the vault entitles you to voting power.

Like in Compound, a user needs to have a certain amount of voting power (called a quorum) to introduce a governance proposal. Unlike in Compound, these quorums are configurable. For example, one could require that most proposals require 100 voting power, but require 200 if they call any of the functions on a specific contract. These are adjusted through *[setCustomQuorum](https://github.com/element-fi/council/blob/0582395e6d720ae4b907bfda4ecd288e80794052/contracts/CoreVoting.sol#L316-L326)*.

The Governance Steering Committee (GSC), is a special voting vault with the ability to introduce proposals without any voting power ([src1](https://github.com/element-fi/council/blob/0582395e6d720ae4b907bfda4ecd288e80794052/contracts/CoreVoting.sol#L120), [src2](https://github.com/element-fi/council/blob/0582395e6d720ae4b907bfda4ecd288e80794052/contracts/CoreVoting.sol#L189-L191)). Although there is reference to the GSC in *CoreVoting*, it is not so tightly integrated that a GSC is forced upon the deployer. I.e., *FIAT doesn't need to use a GSC* if it doesn't want.

### vFDT Migration

As of 04/13/22, 74.5M FDT tokens are staked by 950 stakers. This means that a seamless migration is a must. 

The current system is shown here:
![Comitium Architecture Diagram](https://user-images.githubusercontent.com/45110941/141505843-b611ee45-c3a5-457e-a998-6f6fa84052b3.png)

At a high level, the options for migrating these users are:
1. **Upgrading Comitium to become a voting vault (recommended)**: users' FDT aren't moved, and instead Comitium is adapted to Element's system.
2. **Creating a new 've' voting vault**: FIAT creates a new contract called something like *VoteEscrowedLockingVault*, and users' FDT are moved into the new contract.

In either of these options, there would likely be some behind-the-scenes migration. This is possible because the Comitium diamond is upgradeable by the [FIAT DAO multisig](0x441D4FB492D1e8df573873d21a3445C15A607e09). 


#### **1: Upgrading Comitium to become a voting vault**

Comitium already has a function with the following signature:

```
function votingPowerAtTs(address user, uint256 timestamp)
```

Adapting Comitium to become a voting vault would require FIAT to adapt this function to *queryVotePower*. [This example](https://github.com/fiatdao/comitium/pull/8/files), which contains 3 lines of code, shows how this might be done. There is the matter of converting a block number to a timestamp, but this doesn't seem to be a difficult engineering feat.

A step-by-step overview of how this could look like:
1. Write code that adapts Comitium (via ComitiumFacet) to *[IVotingVault](https://github.com/element-fi/council/blob/main/contracts/interfaces/IVotingVault.sol)*
2. Run integration tests to ensure that said code cannot be manipulated (e.g., flash loan voting)
3. Upgrade ComitiumFacet in Comitium
4. Deploy the [Timelock](https://github.com/element-fi/council/blob/main/contracts/features/Timelock.sol) contract
5. (optional) Deploy the GSC voting vault
6. Deploy [CoreVoting](https://github.com/element-fi/council/blob/main/contracts/CoreVoting.sol), passing in Comitium's address to the constructor.

It's important to note that the GSC needs to be deployed before CoreVoting, if it is to be deployed at all. This is because the GSC is authorized in the constructor of CoreVoting ([src](https://github.com/element-fi/council/blob/0582395e6d720ae4b907bfda4ecd288e80794052/contracts/CoreVoting.sol#L120)), and nowhere else.

The advantage of this option, and why it's recommended, is that you would get most (all?) of the benefits of a new voting vault, but there would be fewer actions required from both the users and FIAT.

#### **2: Creating a new 've' voting vault**

Alternatively, FIAT could build its own voting vault, separate from Comitium. This contract would likely extend *[AbstractVotingVault](https://github.com/element-fi/council/blob/0582395e6d720ae4b907bfda4ecd288e80794052/contracts/vaults/LockingVault.sol#L10)*, overriding key functions like *deposit*, *withdraw*, and *queryVotePower* to use vote escrow mechanics, similar to those implemented in Comitium.

A step-by-step overview of how this could look like:
1. Write a new contract which implements *IVotingVault*
2. Get contract audited
3. Run integration tests to ensure that said code cannot be manipulated (e.g., flash loan voting)
4. Deploy the new voting vault
5. Deploy the [Timelock](https://github.com/element-fi/council/blob/main/contracts/features/Timelock.sol) contract
6. (optional) Deploy the GSC voting vault
7. Deploy [CoreVoting](https://github.com/element-fi/council/blob/main/contracts/CoreVoting.sol), passing in the new voting vault's address to the constructor.
8. Upgrade Comitium to allow users to withdraw, irrespective of former time locks
9. (optional) upgrade Comitium to allow FIAT to batch withdraw on behalf of all users, to minimize friction from user standpoint
10. (optional) call a function on the new voting vault to batch deposit on behalf of all users, setting the same timelock parameters as existed in Comitium

Option 2 would be a large lift, but could be useful if FIAT wanted to do a soup-to-nuts refactor of its governance (for example, for gas optimization purposes). It would also be possible later, and it's not necessary for FIAT to migrate Comitium to a new voting vault at the same time that it upgrades its governance contracts. The two are fairly independent.
