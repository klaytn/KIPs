---
kip: 81
title: Implementing the on-chain governance voting method
author: Yeri (@yeri-lee), Daniel (@dcground), Aidan (@aidan-kwon), Ollie (@blukat29), Eddie<eddie.kim0@klaytn.foundation>
discussion-to: https://govforum.klaytn.foundation/t/kgp-1-implementing-the-on-chain-governance-voting-method/18
status: Final
type: Standards Track
category: Core
created: 2022-09-19
---

## Simple Summary
Introducing a new governance voting method based on the staking amount and implementation of the Klaytn Square, a governance portal

## Abstract
Klaytn introduces a stake-based governance model that provides voting power to governance participants. Currently, one vote per Governance Council(GC) member was cast. The new method will introduce the voting right that will be exercised based on the staking amount with the maximum cap to prevent an entity from making an arbitrary decision. The voting agenda is determined through discussion on the Klaytn Governance Forum, and the topic will be presented and voted on at the newly launched Governance Portal, Klaytn Square. This voting mechanism aims to provide responsibility and obligation of voting to Governance Councils.

## Motivation
The new on-chain voting mechanism discloses GC’s opinion transparently through the portal, allowing anyone to view the result. The governance agenda discussed in Klaytn Governance Forum will be voted on the governance portal, Klaytn Square. Since the voting power is calculated based on the staking amount, this voting process provides more power to GC members who share the same interest as Klaytn by staking and locking up more KLAYs.

## Specification
Klaytn Governance Forum is to freely propose and discuss Klaytn governance items. Once Klaytn Square, the governance portal, opens, the on-chain voting will be executed based on the discussion held in this forum.

Klaytn Square includes the following functions:
- Ability to vote on a governance agenda and view the voting process
- Information about Governance Councils: description, contract address, notice, staking status, staking and reward history, etc.
- View the history of governance agenda and Governance Councils

![voting process diagram](../assets/kip-81/voting_process_diagram.png)

The foundation will provide 7 days of preparation period for voting, providing a time for GC to adjust the staking amount. With the start of the voting, the foundation will announce the list of GC members and their voting power. GC will have 7 days of the voting period.

The Klaytn governance voting system is designed based on the following fundamental premises.
- Klaytn’s major decision-making process should reflect the opinions of as many participants as possible from the ecosystem.
- Participants will be more likely to make a decision that is beneficial to the Klaytn ecosystem if they hold more KLAY. This is based on the premise that the growth of Klaytn’s ecosystem is correlated to the rise in the value of KLAY.
- The governance system should be able to manage the situations in which a particular entity makes an arbitrary decision. This is because the entity may weaken the sustainability of the entire network.
- The act of voting and the gain of voting power is different.

Recognizing a tendency that the member with more KLAY makes a decision that benefits the Klaytn ecosystem in the long run, we are granting a higher voting power to members with more KLAY. Yet, to prevent a particular subject from making an arbitrary decision, we are placing a cap on the voting power one can receive at the maximum. Therefore, a GC member will receive 1 vote per a certain amount of staked KLAY (initial configuration: 5 million KLAY). The maximum number of votes a member can own is one less than the total number of GC members. In other words, [Maximum Voting Power =  Total number of GC members - 1]. For example, if there are 35 GC members, one can own a maximum of 34 voting power. This cap structure prevents potential monopolies.

### Smart Contracts Overview

The Klaytn on-chain governance voting will be conducted on smart contracts. Several contracts and accounts interact together in the process. The below diagram shows the relationship between contracts and accounts.

Find an example implementation [here](https://github.com/klaytn/governance-contracts-audit).

- Contracts
  - **AddressBook**: an existing contract that stores the list of GC nodes, their staking contracts, and their reward recipient addresses.
  - **CnStakingV2**: an updated version of the existing CnStakingContract. GCs stake their KLAYs to earn rights to validate blocks and cast on-chain votes.
  - **StakingTracker**: a new contract that tracks voting-related data from AddressBook and CnStakingV2 contracts.
  - **Voting**: a new contract that processes the on-chain voting. It stores governance proposals, counts votes, and sends approved transactions.
- Accounts
  - **AddressBook admins**: a set of accounts controlled by the Foundation which can manage the list of GCs in the AddressBook. AddressBook admins are managed in the AddressBook contract.
  - **CnStaking admins**: a set of accounts controlled by each GC that can manage the staked KLAYs and its voter account. CnStaking admins are managed in the respective CnStakingV2 contracts. Note that every CnStakingV2 contract has a different set of admins.
  - **Voter account**: an account controlled by each GC member that can cast on-chain votes. But this account cannot withdraw KLAYs from CnStakingV2 contracts. A Voter account is appointed at CnStakingV2 contracts of the respective GC member.
  - **Secretariat account**: an account controlled by the Foundation which can propose and execute on-chain governance proposals. The secretariat account is managed in the Voting contract.

![contracts and accounts](../assets/kip-81/smart_contract_relation.png)

### AdressBook

In Cypress mainnet and Baobab testnet, an [AddressBook contract](https://github.com/klaytn/klaytn/blob/v1.9.1/contracts/reward/contract/AddressBook.sol) is deployed at address `0x0000000000000000000000000000000000000400`. For the purpose of Voting, the following function of the AddressBook is used.

```solidity
interface IAddressBook {
    /// @dev Returns all addresses in the AddressBook.
    ///
    /// Each GC's addresses are stored in the same index of the three arrays.
    /// i.e. GC[i] is described in (nodeIds[i], stakingContracts[i], rewardAddresses[i])
    /// However, a GC may operate multiple staking contracts. In this case,
    /// the GC occupies multiple indices with the same reward address.
    ///
    /// @return nodeIds           GC consensus node address list
    /// @return stakingContracts  GC staking contract address list
    /// @return rewardAddresses   GC reward recipient address list
    /// @return pocAddress        PoC(KGF) recipient address
    /// @return kirAddress        KIR recepient address
    function getAllAddressInfo() external view returns(
        address[] memory nodeIds,
        address[] memory stakingContracts,
        address[] memory rewardAddresses,
        address pocAddress,
        address kirAddress);
}
```

### CnStakingV2

CnStakingV2 is an upgraded version of [CnStakingContract](https://github.com/klaytn/klaytn/blob/v1.9.1/contracts/cnstaking/CnStakingContract.sol). The CnStakingContract has been serving the purpose of locking GC stakes. V2 will add two new features related to StakingTracker, which will be described later.

- It holds a new unique identifier "GC ID" which is a nonzero integer assigned to each GC member. If a GC member owns multiple CnStakingV2 contracts, the contracts must contain the same GC ID value.
- Whenever its KLAY balance changes, V2 notifies the StakingTracker.
- V2 stores a voter account address, and notifies the StakingTracker whenever it changes. The voter account can be changed by its admins.

To support the new features, the following functions are added. The functions `setStakingTracker` and `setGCId` is only callable before contract [initialization](https://github.com/klaytn/klaytn/blob/v1.9.1/contracts/cnstaking/CnStakingContract.sol#L237). Functions `updateStakingTracker` and `updateVoterAddress` are invoked upon approval of CnStaking admins using the existing [CnStaking multisig facility](https://github.com/klaytn/klaytn/blob/v1.9.1/contracts/cnstaking/CnStakingContract.sol#L597).

```solidity
abstract contract CnStakingV2 {
    event UpdateStakingTracker(address stakingTracker);
    event UpdateVoterAddress(address voterAddress);
    event UpdateGCId(uint256 gcId);

    function stakingTracker() external view returns(address);
    function voterAddress() external view returns(address);
    function gcId() external view returns(uint256);

    /// @dev Set the initial stakingTracker address
    /// Emits an UpdateStakingTracker event.
    function setStakingTracker(address _tracker) external beforeInit;

    /// @dev Set the gcId
    /// The gcId never changes once initialized.
    /// Emits an UpdateGCId event.
    function setGCId(uint256 _gcId) external beforeInit;

    /// @dev Update the StakingTracker address this contract reports to
    /// Emits an UpdateStakingTracker event.
    function submitUpdateStakingTracker(address _tracker) external;
    function updateStakingTracker(address _tracker) external onlyMultisigTx;

    /// @dev Update the voter address of this GC
    /// Emits an UpdateVoterAddress event.
    function submitUpdateVoterAddress(address _addr) external;
    function updateVoterAddress(address _addr) external onlyMultisigTx;
}
```

#### Notifying changes to StakingTracker

To notify balance and voter account changes to the StakingTracker contract, the CnStakingV2 contract shall call the StakingTracker whenever after every change. For example,

```solidity
abstract contract CnStakingV2 {
    function depositLockupStakingAndInit() payable beforeInit() {
        // ...
        stakingTracker.refreshStake(address(this));
    }
    function stakeKlay() payable {
        // ...
        stakingTracker.refreshStake(address(this));
    }
    function withdrawLockupStaking(address _to, uint256 _value) {
        // ...
        _to.transfer(_value);
        stakingTracker.refreshStake(address(this));
    }
    function withdrawApprovedStaking(uint256 _approvedWithdrawalId) {
        // ...
        _to.transfer(_value);
        stakingTracker.refreshStake(address(this));
    }
    function updateVoterAddress(address _addr) {
        // ...
        voterAddress = _addr;
        stakingTracker.refreshVoter(address(this));
    }
}

interface IStakingTracker {
    /// @dev Re-evaluate Tracker contents related to the staking contract
    /// @param staking  The CnStaking contract address
    function refreshStake(address staking) external;

    /// @dev Update a GC's voter address from the given address
    /// @param staking  The CnStaking contract address
    function refreshVoter(address staking) external;
}
```

#### Version identifier

To distinguish the CnStakingContract and CnStakingV2 contracts, V2 will have VERSION value 2.

```solidity
abstract contract CnStakingV2 {
    uint256 constant public VERSION = 2;
}
```

#### Example implementation

See [here](https://github.com/klaytn/governance-contracts-audit/blob/main/contracts/CnStakingV2.sol) for an example implementation.

### StakingTracker

StakingTracker collects voting-related data of each GC member.

- Staking amounts and voting powers \
Stored separately in `Tracker` structs. Each `Tracker` struct can be updated between `trackStart` and `trackEnd` blocks, but becomes immutable afterward, essentially freezing the voting powers. A `Tracker` struct is first created in `createTracker()`, and updated by `refreshStake()`.

- Each GC’s voter accounts \
Stored in global mappings, and can be updated at any time. Voter account mappings are updated by `refreshVoter()`.

#### Contract interface

```solidity
abstract contract StakingTracker {
    struct Tracker {
        // Tracked block range.
        // Balance changes are only updated if trackStart <= block.number < trackEnd.
        uint256 trackStart;
        uint256 trackEnd;

        // Determined at createTracker() and does not change.
        uint256[] gcIds;
        mapping(uint256 => bool) gcExists;
        mapping(address => uint256) stakingToGCId;

        // Balances and voting powers.
        // First collected at createTracker() and updated at refreshStake() until trackEnd.
        mapping(address => uint256) stakingBalances; // staking address balances
        mapping(uint256 => uint256) gcBalances; // consolidated GC balances
        mapping(uint256 => uint256) gcVotes; // GC voting powers
        uint256 totalVotes;
        uint256 numEligible;
    }

    // Tracker objects. Added by createTracker().
    mapping(uint256 => Tracker) private trackers; // trackerId => Tracker
    uint256[] private allTrackerIds;

    // 1-to-1 mapping between nodeId and voter account.
    // Updated by refreshVoter().
    mapping(uint256 => address) public gcIdToVoter;
    mapping(address => uint256) public voterToGCId;

    // Minimum staking amount to have 1 vote
    function MIN_STAKE() external view returns(uint256);

    /// @dev Creates a new Tracker and populate initial values from AddressBook
    function createTracker(uint256 trackStart, uint256 trackEnd)
        external returns(uint256 trackerId);

    /// @dev Re-evaluate Tracker contents related to the staking contract
    /// Anyone can call this function, but `staking` must be a staking contract
    /// registered to the AddressBook.
    function refreshStake(address staking) external;

    /// @dev Re-evaluate voter account mapping related to the staking contract
    /// Anyone can call this function, but `staking` must be a staking contract
    /// registered to the AddressBook.
    /// Note that two different nodes cannot appoint the same voter address.
    function refreshVoter(address staking) external;

    /// @dev Return integer fields of a tracker
    function getTrackerSummary(uint256 trackerId) external view returns(
        uint256 trackStart,
        uint256 trackEnd,
        uint256 numGCs,
        uint256 totalVotes,
        uint256 numEligible)

    /// @dev Return balances and votes of all GC members stored in a tracker
    function getAllTrackedGCs(uint256 trackerId) external view returns(
        uint256[] memory gcIds,
        uint256[] memory gcBalances,
        uint256[] memory gcVotes)

    /// @dev Return the balance and votes of a specified GC member
    function getTrackedGC(uint256 trackerId, uint256 gcId) external view returns(
        uint256 gcBalance,
        uint256 gcVotes)
}
```

#### Usage

A voting contract can utilize StakingTracker as follows.

1. When a governance proposal is submitted, the voting contract calls `createTracker()` to finalize the eligible GC members list and evaluate voting powers.
2. Before `trackEnd` block, GC members stake or unstake their KLAYs from their CnStakingV2 contracts. The CnStakingV2 contracts will then call `refreshStake()` to notify the balance change.
3. GC members may change their voter account in their CnStakingV2 contracts. The CnStakingV2 contracts will then call `refreshVoter()` to notify voter account change.
4. After `trackEnd` block, the voting powers are frozen. The voting contract uses StakingTracker getters to process votes cast by voter accounts.

#### Example implementation

See [here](https://github.com/klaytn/governance-contracts-audit/blob/main/contracts/StakingTracker.sol) for an example implementation.

### Voting

The Voting contract operates the on-chain governance votes. The Voting contract stores governance proposals, counts casted votes, and sends on-chain transactions. Its design is influenced by [Compound Finance](https://docs.compound.finance/v2/governance/) and [OpenZeppelin](https://docs.openzeppelin.com/contracts/4.x/api/governance).

A proposal contains textual descriptions and optionally on-chain transactions. The transactions are executed on behalf of the voting contract if the proposal receives enough votes.

While GCs hold their voting rights from staked KLAYs, a special secretariat account manages the overall on-chain governance process. Governance proposals can be submitted by the secretariat account, or any GC if the secretariat account is not set. The secretariat account can be updated by on-chain governance.

#### Voting steps overview

Under the default timing settings, a typical governance proposal is handled as follows.

1. When a governance proposal is submitted, it enters a 7-day preparation period where GCs can adjust their voting powers by staking or unstaking KLAYs. The list of voters (i.e., GCs) is finalized at the moment of proposal submission. However, voting powers are finalized at the end of the preparation period.
2. A 7-day voting period immediately follows after the preparation period.
3. If there are enough Yes votes, the transactions can be queued within 14 days after voting ends.
4. The transaction is delayed by 2 days to be executable. This delay gives the ecosystem enough time to adjust to the change and perform a final review of transactions.
5. After the delay, the transaction can be executed within 14 days.

The proposer of a proposal can cancel it at any time prior to execution. If a passed transaction is not queued or executed within the timeout, the proposal automatically expires.

![voting steps](../assets/kip-81/voting_steps.png)

#### Access control

Voting contract has two category of users, secretary and voters.

- The secretary is a single account stored in the `secretary` variable. This account is intended to be controlled by the Klaytn Foundation. It will serve the GC by assisting administrative works such as submitting and executing proposals.
- The voters are identified by their GC ID. The list of voters differs per proposal, depending on the list of GC members registered in AddressBook and their staking amounts at the time of proposal submission.

In its default setting, only the secretary can propose and execute proposals. Later the voters may be allowed to propose and execute proposals by changing the following access rule.

| Name               | Meaning                                                                               | Default Value |
| ---                | ---                                                                                   | ---           |
| `secretaryPropose` | True if the secretary can propose()                                                   | true          |
| `voterPropose`     | True if any eligible voter at the time of the submission can propose() a proposal     | false         |
| `secretaryExecute` | True if the secretary can queue() and execute()                                       | true          |
| `voterExecute`     | True if any eligible voter of a given proposal can queue() and execute() the proposal | false         |

#### Timing rule

The proposal timeline is dictated by several timing values below. The settings are expressed in
block numbers. The number of days in the below table is calculated based on 1 block/sec assumption.

To support flexible operation, Voting contract allows each proposal to have different `votingDelay` and `votingPeriod` within respective min and max values.
However, `queueTimeout`, `execDelay` and `execTimeout` are should not be configurable.

| Name              | Meaning                                                      | Default Value     | Configurability        |
|-------------------|--------------------------------------------------------------|-------------------|------------------------|
| `minVotingDelay`  | Minimum `votingDelay`                                        | 86400 (1 day)     | via `updateTimingRule` |
| `maxVotingDelay`  | Maximum `votingDelay`                                        | 2419200 (28 days) | via `updateTimingRule` |
| `minVotingPeriod` | Minimum `votingPeriod`                                       | 86400 (1 day)     | via `updateTimingRule` |
| `maxVotingPeriod` | Maximum `votingPeriod`                                       | 2419200 (28 days) | via `updateTimingRule` |
| `votingDelay`     | Delay from proposal submission to voting start               |                   | Specify per proposal   |
| `votingPeriod`    | Duration of the voting                                       |                   | Specify per proposal   |
| `queueTimeout`    | Grace period to queue() passed proposals                     | 1209600 (14 days) | Cannot modify          |
| `execDelay`       | A minimum delay before a queued transaction can be executed  | 172800 (2 days)   | Cannot modify          |
| `execTimeout`     | Grace period to execute() queued proposals since `execDelay` | 1209600 (14 days) | Cannot modify          |

#### Quorum

A proposal passes when it satisfies a combination of the following conditions.

- `CountQuorum` = At least 1/3 of all eligible voters cast votes
- `PowerQuorum` = At least 1/3 of all eligible voting powers cast votes
- `MajorityYes` = Yes votes are more than half of total cast votes (i.e. `Yes > No + Abstain`)
- `PassCondition = (CountQuorum or PowerQuorum) and MajorityYes`

#### Contract interface

```solidity
interface Voting {
    enum ProposalState { Pending, Active, Canceled, Failed, Passed, Queued, Expired, Executed }
    enum VoteChoice { No, Yes, Abstain }
    struct Receipt {
        bool hasVoted;
        uint8 choice;
        uint256 votes;
    }

    // events

    /// @dev Emitted when a proposal is created
    /// @param signatures  Array of empty strings; for compatibility with OpenZeppelin
    event ProposalCreated(
        uint256 proposalId, address proposer,
        address[] targets, uint256[] values, string[] signatures, bytes[] calldatas,
        uint256 voteStart, uint256 voteEnd, string description);

    /// @dev Emitted when a proposal is canceled
    event ProposalCanceled(uint256 proposalId);

    /// @dev Emitted when a proposal is queued
    /// @param eta  The block number where transaction becomes executable.
    event ProposalQueued(uint256 proposalId, uint256 eta);

    /// @dev Emitted when a proposal is executed
    event ProposalExecuted(uint256 proposalId);

    /// @dev Emitted when a vote is cast
    /// @param reason  An empty string; for compatibility with OpenZeppelin
    event VoteCast(address indexed voter, uint256 proposalId,
                   uint8 choice, uint256 votes, string reason);

    /// @dev Emitted when the StakingTracker is changed
    event UpdateStakingTracker(address oldAddr, address newAddr);

    /// @dev Emitted when the secretary is changed
    event UpdateSecretary(address oldAddr, address newAddr);

    /// @dev Emitted when the AccessRule is changed
    event UpdateAccessRule(
        bool secretaryPropose, bool voterPropose,
        bool secretaryExecute, bool voterExecute);

    /// @dev Emitted when the TimingRule is changed
    event UpdateTimingRule(
        uint256 minVotingDelay,  uint256 maxVotingDelay,
        uint256 minVotingPeriod, uint256 maxVotingPeriod);

    // mutator functions

    /// @dev Create a Proposal
    /// @param description   Proposal text
    /// @param targets       List of transaction target addresses
    /// @param values        List of KLAY values to send along with transactions
    /// @param calldatas     List of transaction calldatas
    /// @param votingDelay   Delay from proposal submission to voting start in block numbers
    /// @param votingPeriod  Duration of the voting in block numbers
    function propose(
        string memory description,
        address[] memory targets,
        uint256[] memory values,
        bytes[] memory calldatas,
        uint256 votingDelay,
        uint256 votingPeriod
    ) external returns (uint256 proposalId);

    /// @dev Cancel a proposal
    /// The proposal must be in Pending state
    /// Only the proposer of the proposal can cancel the proposal.
    function cancel(uint256 proposalId) external;

    /// @dev Cast a vote to a proposal
    /// The proposal must be in Active state
    /// A node can only vote once for a proposal
    /// choice must be one of VoteChoice.
    function castVote(uint256 proposalId, uint8 choice) external;

    /// @dev Queue a passed proposal
    /// The proposal must be in Passed state
    /// Current block must be before `queueDeadline` of this proposal
    /// If secretary is null, any GC with at least 1 vote can queue.
    /// Otherwise only secretary can queue.
    function queue(uint256 proposalId) external;

    /// @dev Execute a queued proposal
    /// The proposal must be in Queued state
    /// Current block must be after `eta` and before `execDeadline` of this proposal
    /// If secretary is null, any GC with at least 1 vote can execute.
    /// Otherwise only secretary can execute.
    function execute(uint256 proposalId) external payable;

    /// @dev Update the StakingTracker address
    /// Should not be called if there is an active proposal
    function updateStakingTracker(address newAddr) external;

    /// @dev Update the secretary account
    function updateSecretary(address newAddr) external;

    /// @dev Update the access rule
    function updateAccessRule(
        bool secretaryPropose, bool voterPropose,
        bool secretaryExecute, bool voterExecute) external;

    /// @dev Update the timing rule
    function updateTimingRule(
        uint256 minVotingDelay,  uint256 maxVotingDelay,
        uint256 minVotingPeriod, uint256 maxVotingPeriod) external;

    // getter functions

    function stakingTracker() external view returns(address);
    function secretary() external view returns(address);
    function accessRule() external view returns(
        bool secretaryPropose, bool voterPropose,
        bool secretaryExecute, bool voterExecute);
    function timingRule() external view returns(
        uint256 minVotingDelay,  uint256 maxVotingDelay,
        uint256 minVotingPeriod, uint256 maxVotingPeriod);
    function queueTimeout() external view returns(uint256);
    function execDelay() external view returns(uint256);
    function execTimeout() external view returns(uint256);

    /// @dev The id of the last created proposal
    /// Retrurns 0 if there is no proposal.
    function lastProposalId() external view returns(uint256);

    /// @dev State of a proposal
    function state(uint256 proposalId) external view returns(ProposalState);

    /// @dev Check if a proposal is passed
    /// Note that its return value represents the current voting status,
    /// and is subject to change until the voting ends.
    function checkQuorum(uint256 proposalId) external view returns(bool);

    /// @dev Resolve the voter account into its gcId and voting powers
    /// Returns the currently assigned gcId. Returns the voting powers
    /// effective at the given proposal. Returns zero gcId and 0 votes
    /// if the voter account is not assigned to any eligible GC.
    ///
    /// @param proposalId  The proposal id
    /// @return gcId    The gcId assigned to this voter account
    /// @return votes   The amount of voting powers the voter account represents
    function getVotes(uint256 proposalId, address voter) external view returns(uint256, uint256);

    /// @dev General contents of a proposal
    function getProposalContent(uint256 proposalId) external view returns(
        uint256 id,
        address proposer,
        string memory description
    );

    /// @dev Transactions in a proposal
    /// @return signatures  Array of empty strings; for compatibility with OpenZeppelin
    function function getActions(uint256 proposalId) external view returns(
        address[] memory targets,
        uint256[] memory values,
        string[] memory signatures,
        bytes[] memory calldatas
    );

    /// @dev Timing and state related properties of a proposal
    function getProposalSchedule(uint256 proposalId) external view returns(
        uint256 voteStart,      // propose()d block + votingDelay
        uint256 voteEnd,        // voteStart + votingPeriod
        uint256 queueDeadline,  // voteEnd + queueTimeout
        uint256 eta,            // queue()d block + execDelay
        uint256 execDeadline,   // queue()d block + execDelay + execTimeout
        bool canceled,          // true if successfully cancel()ed
        bool queued,            // true if successfully queue()d
        bool executed           // true if successfully execute()d
    );

    /// @dev Vote counting related properties of a proposal
    function getProposalTally(uint256 proposalId) external view returns(
        uint256 totalYes,
        uint256 totalNo,
        uint256 totalAbstain,
        uint256 quorumCount,      // required number of voters
        uint256 quorumPower,      // required number of voting powers
        uint256[] memory voters,  // list of gcIds who voted
    )

    /// @dev Individual vote receipt
    function getReceipt(uint256 proposalId, uint256 gcId)
        external view returns(Receipt memory);
}
```

#### Fetching votes from StakingTracker

Calculating votes requires information from StakingTracker contract. The Voting contract shall utilize the getters to find the voting powers of each voter as well as quorum conditions. Below is an example implementation of `castVote()` and `checkQuorum()` functions.

```solidity
abstract contract Voting {
    struct Proposal {
        uint256 trackerId;  // StakingTracker trackerId created in propose()
        uint256 totalYes;
        uint256 totalNo;
        uint256 totalAbstain;
        uint256[] voters;   // List of gcIds
        mapping(uint256 => Receipt) receipts;
    }
    mapping(uint256 => Proposal) proposals;
    IStakingTracker stakingTracker;

    function castVote(uint256 proposalId, uint8 choice) external {
        Proposal storage p = proposals[proposalId];

        (uint256 gcId, uint256 votes) = getVotes(proposalId, msg.sender);
        require(gcId != 0, "Not a registered voter");
        require(votes > 0, "Not eligible to vote");

        require(choice == uint8(VoteChoice.Yes) ||
                choice == uint8(VoteChoice.No) ||
                choice == uint8(VoteChoice.Abstain), "Not a valid choice");
        require(!p.receipts[gcId].hasVoted, "Already voted");

        p.receipts[gcId].hasVoted = true;
        p.receipts[gcId].choice = choice;
        p.receipts[gcId].votes = votes;
        if (choice == VoteChoice.Yes) { p.totalYes += votes; }
        if (choice == VoteChoice.No) { p.totalNo += votes; }
        if (choice == VoteChoice.Abstain) { p.totalAbstain += votes; }
        p.voters.push(gcId);
    }

    function getVotes(uint256 proposalId, address voter) public view returns(
        uint256 gcId, uint256 votes) {
        Proposal storage p = proposals[proposalId];

        gcId = IStakingTracker(p.stakingTracker).voterToGCId(voter);
        ( , votes) = IStakingTracker(p.stakingTracker).getTrackedGC(p.trackerId, gcId);
    }

    function checkQuorum(uint256 proposalId) external view returns(bool) {
        Proposal storage p = proposals[proposalId];

        (uint256 quorumCount, uint256 quorumPower) = getQuorum(proposalId);
        uint256 totalVotes = p.totalYes + p.totalNo + p.totalAbstain;
        uint256 quorumYes = p.totalNo + p.totalAbstain + 1; // more than half of all votes

        bool countPass = (p.voters.length >= quorumCount);
        bool powerPass = (totalVotes      >= quorumPower);
        bool approval  = (p.totalYes      >= quorumYes);

        return ((countPass || powerPass) && approval);
    }

    /// @dev Calculate count and power quorums for a proposal
    function getQuorum(uint256 proposalId) private view returns(
        uint256 quorumCount, uint256 quorumPower) {

        Proposal storage p = proposals[proposalId];
        if (p.quorumCount != 0) { // return cached numbers
            return (p.quorumCount, p.quorumPower);
        }

        ( , , , uint256 totalVotes, uint256 numEligible) =
            IStakingTracker(p.stakingTracker).getTrackerSummary(p.trackerId);

        quorumCount = (numEligible + 2) / 3; // more than or equal to 1/3 of all GC members
        quorumPower = (totalVotes + 2) / 3; // more than or equal to 1/3 of all voting powers
        return (quorumCount, quorumPower);
    }
}
```

#### Determining state

The state of a proposal can be determined from stored variables, vote counts, and current block number. Below is an example implementation of `state()` function.

```solidity
abstract contract Voting {
    struct Proposal {
        uint256 voteStart;      // propose()d block + votingDelay
        uint256 voteEnd;        // voteStart + votingPeriod
        uint256 queueDeadline;  // voteEnd + queueTimeout
        uint256 eta;            // queue()d block + execDelay
        uint256 execDeadline;   // queue()d block + execDelay + execTimeout
        bool canceled;          // true if successfully cancel()ed
        bool queued;            // true if successfully queue()d
        bool executed;          // true if successfully execute()d
    }
    mapping(uint256 => Proposal) proposals;

    function state(uint256 proposalId) external view returns (ProposalState) {
        Proposal storage p = proposals[proposalId];

        if (p.executed) {
            return ProposalState.Executed;
        } else if (p.canceled) {
            return ProposalState.Canceled;
        } else if (block.number < p.voteStart) {
            return ProposalState.Pending;
        } else if (block.number <= p.voteEnd) {
            return ProposalState.Active;
        } else if (!checkQuorum(proposalId)) {
            return ProposalState.Failed;
        }

        if (!p.queued) {
            // A proposal without an action does not expire
            if (block.number <= p.queueDeadline || p.targets.length == 0) {
                return ProposalState.Passed;
            } else {
                return ProposalState.Expired;
            }
        } else {
            if (block.number <= p.execDeadline) {
                return ProposalState.Queued;
            } else {
                return ProposalState.Expired;
            }
        }
    }
}
```

#### Executing transactions

The transactions in a passed proposal are executed in the order they appear in the `propose()` function arguments.  Each transaction is sent to `targets[i]`, carrying `values[i]` of KLAYs, with `calldatas[i]` as input data. Below is an example of `execute()` function.

```solidity
abstract contract Voting {
    struct Proposal {
        address[] targets;
        uint256[] values;
        bytes[] calldatas;

    }
    mapping(uint256 => Proposal) proposals;

    function execute(uint256 proposalId) external payable {
        Proposal storage p = proposals[proposalId];

        for (uint256 i = 0; i < p.targets.length; i++) {
            (bool success, bytes memory result) =
                    p.targets[i].call{value: p.values[i]}(p.calldatas[i]);
            require(success, "Transaction failed");
        }
    }
}
```

#### Example implementation

See [here](https://github.com/klaytn/governance-contracts-audit/blob/main/contracts/Voting.sol) for an example implementation.

## Rationale

#### CnStakingV2 and StakingTracker instead of standard ERC20Votes

It is common for DAOs to manage their governance tokens in an [ERC20Votes](https://docs.openzeppelin.com/contracts/4.x/api/token/erc20#ERC20Votes) contract. However new contracts CnStakingV2 and StakingTracker were used to manage staked KLAYs. Using the new contracts provides better compatibility with ecosystem Dapps like public staking services. It also does not break the existing block validator selection algorithm that depends on AddressBook.

#### StakingTracker refreshStake and refreshVoting has no access control

It is a remnant of a previous design. Those functions were originally intended to allow optionally or gradually migrate from CnStakingContract to CnStakingV2 contracts. Now that old CnStakingContract are excluded from the voting power calculation, such a feature is unnecessary but the refreshStake and refreshVoter functions retain its shape.

#### Separate voter account

In a security standpoint, it is safer to have separate accounts for different roles. Existing CnStakingV2 admins take a financial role - withdrawing staked KLAYs - and the voter account take a voting role only.

#### Proposals have expired state

Proposal transactions expire if they are not queued or executed within the deadline. This guarantees the timeliness of the transactions. If a proposal was left unexecuted for a long time, the proposal might be irrelevant anymore and require a fresh proposal.

## Expected Effect

The proposed GC Voting method is expected to produce the following changes:
- All members in the Klaytn ecosystem grow together with credibility
- Providing obligation and authority to GC motivates active participation in governance voting activities and the KLAY staking
- Anyone can easily view the proposals and voting status through the governance portal. It encourages holders to participate and give responsibility and authority to GCs.
- The Klaytn network can take a step closer to transparency and decentralized networks.

## Backward Compatibility

- GCs should deploy new CnStakingV2 contracts and move their staked KLAYs. Existing CnStakingContract (i.e. V1) balances are excluded from voting power calculation.
- However, block validator selection is not affected by the change, so GCs can still participate in consensus before migrating to CnStakingV2.

## Reference
n/a

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
