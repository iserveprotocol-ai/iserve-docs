# Governance FAQ

## Table of Contents
1. [General Questions](#general-questions)
2. [Token Questions](#token-questions)
3. [Voting Questions](#voting-questions)
4. [Proposal Questions](#proposal-questions)
5. [Delegation Questions](#delegation-questions)
6. [Technical Questions](#technical-questions)
7. [Security Questions](#security-questions)

## General Questions

### What is iServe Protocol governance?
iServe Protocol governance is a decentralized system that allows $ISERVE-GOV token holders to participate in protocol decision-making. Token holders can vote on proposals, delegate their voting power, and participate in shaping the future of the protocol.

### Who can participate in governance?
Anyone who holds $ISERVE-GOV tokens can participate in governance. There are different levels of participation:
- **Token Holders**: Can vote on proposals and delegate voting power
- **Delegates**: Can receive delegated voting power and vote on behalf of delegators
- **Proposal Creators**: Need 10M+ tokens to create proposals
- **Community Members**: Can participate in discussions without holding tokens

### How do I get started with governance?
1. Acquire $ISERVE-GOV tokens through approved exchanges
2. Set up a compatible Web3 wallet (MetaMask, WalletConnect, etc.)
3. Delegate your voting power (to yourself or a trusted representative)
4. Join the governance forum and communication channels
5. Start participating in discussions and voting on proposals

### Where can I find governance information?
- **Governance Portal**: [governance.iserveprotocol.com](https://governance.iserveprotocol.com)
- **Forum**: [forum.iserveprotocol.com](https://forum.iserveprotocol.com)
- **Discord**: [discord.gg/iserve](https://discord.gg/iserve) (#governance channel)
- **Documentation**: This documentation set
- **Weekly Calls**: Tuesdays 15:00 UTC

## Token Questions

### What is $ISERVE-GOV?
$ISERVE-GOV is the governance token of the iServe Protocol. It's an ERC-20 token that provides voting rights and enables participation in protocol governance.

### What's the total supply of $ISERVE-GOV?
The total supply is 1,000,000,000 (1 billion) tokens, distributed as follows:
- **Ecosystem & Community**: 40% (400M tokens)
- **Core Contributors**: 20% (200M tokens)
- **Treasury**: 15% (150M tokens)
- **Investors**: 15% (150M tokens)
- **Public Sale & Liquidity**: 10% (100M tokens)

### Can I earn rewards with my tokens?
Yes, there are several ways to earn rewards:
- **Staking**: Stake tokens to earn rewards and enhanced voting power
- **Governance Participation**: Active governance participants may receive rewards
- **Liquidity Provision**: Provide liquidity to earn trading fees (external DEXs)

## Voting Questions

### How does voting work?
Voting follows this process:
1. **Proposal Creation**: Someone with 10M+ tokens creates a proposal
2. **Voting Delay**: ~1 day delay before voting begins
3. **Voting Period**: ~7 days for casting votes
4. **Vote Counting**: Votes are tallied (For/Against/Abstain)
5. **Execution**: If passed and after timelock, proposal is executed

### What voting options do I have?
For each proposal, you can vote:
- **For**: Support the proposal
- **Against**: Oppose the proposal
- **Abstain**: Participate without taking a position (counts toward quorum)

### How much voting power do I have?
Your voting power equals:
- Your token balance at the time of proposal creation
- Plus any tokens delegated to you
- Plus any staking multipliers (if applicable)

### What is quorum and why does it matter?
Quorum is the minimum participation required for a vote to be valid. Currently set at 4% of total supply (40M tokens). If quorum isn't reached, the proposal fails regardless of vote distribution.

## Proposal Questions

### Who can create proposals?
Anyone holding at least 10,000,000 $ISERVE-GOV tokens (1% of total supply) can create governance proposals.

### What types of proposals can be created?
- **Parameter Changes**: Modify governance or protocol parameters
- **Treasury Allocation**: Allocate treasury funds for specific purposes
- **Protocol Upgrades**: Deploy new smart contract versions
- **Emergency Actions**: Respond to security incidents (expedited process)
- **Grants**: Fund ecosystem development projects
- **Partnerships**: Approve strategic partnerships

### How do I create a proposal?
1. **Pre-proposal Discussion**: Engage community in governance forum
2. **Draft Proposal**: Use appropriate template from our documentation
3. **Community Review**: Incorporate feedback and suggestions
4. **Technical Review**: Get security/technical review if needed
5. **Submit On-chain**: Submit through governance portal or smart contract
6. **Campaign**: Advocate for your proposal during voting period

## Delegation Questions

### What is delegation?
Delegation allows you to transfer your voting power to another address (delegate) while still owning your tokens. The delegate can then vote on your behalf.

### Why should I delegate?
Delegation is useful if you:
- Don't have time to research every proposal
- Want your tokens to participate in governance consistently
- Trust someone else's judgment on governance matters
- Want to support active governance participants

### How do I delegate my tokens?
You can delegate through:
- **Governance Portal**: Most user-friendly option
- **Smart Contract**: Direct interaction with governance token contract
- **Wallet Interfaces**: Some wallets support delegation

## Technical Questions

### Which networks is governance deployed on?
iServe Protocol governance is deployed on:
- **Ethereum Mainnet**: Primary governance deployment
- **Testnets**: For testing and development (Goerli, Sepolia)

### What smart contracts are involved in governance?
- **IServeGovToken**: The governance token contract
- **IServeGovernance**: Main governor contract (OpenZeppelin Governor)
- **IServeTimelock**: Timelock controller for execution delays
- **GovernanceIntegration**: Protocol-specific governance features
- **GovernanceParameters**: Configurable governance parameters

### Can governance parameters be changed?
Yes, governance parameters can be updated through the standard governance process. Parameter changes have built-in safety bounds to prevent dangerous configurations.

## Security Questions

### How is governance protected from attacks?
Multiple security measures protect governance:
- **Timelock Delays**: 2-7 day delays before execution
- **Parameter Bounds**: Prevents dangerous parameter changes
- **Flash Loan Protection**: Uses historical token balances for voting
- **Emergency Multisig**: Can pause in emergency situations
- **Proposal Validation**: Automatic checks for valid proposals

### What is the emergency multisig?
The emergency multisig is a 5-of-9 multisig wallet controlled by trusted community members. It can:
- Pause protocol contracts in emergencies
- Make emergency parameter changes within defined bounds
- Coordinate incident response
- All actions must be ratified by governance within 7 days

### What should I do if I find a security issue?
If you find a security issue:
1. **Don't Disclose Publicly**: Keep the issue confidential initially
2. **Contact Security Team**: Email security@iserveprotocol.com
3. **Provide Details**: Include technical details and potential impact
4. **Coordinate Disclosure**: Work with team on responsible disclosure
5. **Bug Bounty**: May be eligible for bug bounty rewards

---

*This FAQ is maintained by the iServe Protocol community. If you have additional questions, please ask in the governance forum or Discord.*