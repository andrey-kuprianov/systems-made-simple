---
layout: post
title: "The Rise of Model Checker: Verifying Blockchain Monitors In and Near Realtime"
date: 2024-07-07
categories: 
  - "solarkraft"
tags: 
  - "apalache"
  - "formal-methods"
  - "runtime-monitor"
  - "smart-contracts"
  - "solarkraft"
  - "soroban"
  - "specification"
  - "stellar"
  - "tla"
  - "tlaplus"
---

![]({{ site.baseurl }}/img/solarkraft.png)

_Solarkraft has been developed in collaboration by [Igor Konnov][], [Jure Kukovec][], [Andrey Kuprianov][] and [Thomas Pani][]._

_This is the fifth and last in a series of blog posts introducing [Solarkraft][], a TLA+-based runtime monitoring solution for [Soroban smart contracts][Soroban]. The first post,_ ["A New Hope – Why Smart Contract Bugs Matter and How Runtime Monitoring Saves the Day"][part1] _gives an overview of smart contracts, explains how traditional security fails to address major challenges in securing crypto assets, and introduces runtime monitoring as a solution. The second post,_ ["Guardians of the Blockchain: Small and Modular Runtime Monitors in TLA+ for Soroban Smart Contracts"][part2] _introduces the language of Solarkraft monitors. The third post,_ ["How to Run Solarkraft"][part3] _gives an overview of the various features of Solarkraft, and explains how to use each one, step-by-step. The forth post, _ ["The Force Awakens: Hybrid Blockchain Runtime Monitors"][part4] _explores the distinctions between direct and reverse blockchain monitors._

In this post we first formally define what are hybrid blockchain runtime monitors (from the formal methods point of view), as then proceed to explore the far-reaching avenues of how to go from _offline monitoring_, as done now in Solarkraft, to truly _online monitoring_ on the live blockchain.


## Verifying Blockchain Monitors

All that is nice and good, but there are a few questions that still need to be addressed, as people with different backgrounds might feel:

- A formal methods person: "How do you _verify_ monitor specs? What are your verification conditions?"
- A math person: "What about verification _complexity_?"
- A software engineering person: "How do you _practically check_ them on the live blockchain?"

This section outlines the answers to the above questions. TL;DR; for those who are not interested in these details:

- We verify monitor specs via a) producing verification conditions from each monitor specification; b) extracting transactions from the blockchain; c) validating each transaction against verification conditions using the Apalache model checker.
- Complexity of verifying monitor specs is _linear_ wrt. the number of conditions in the specification, and the number of transactions: each condition is checked _at most once_ against every transaction (but many checks may be skipped/optimized away).
- Practically, we integrate monitor specs as outlined here in the `solarcraft verify` command; the documentation for which can be found elsewhere in this repo. `Solarcraft` is a tool that we write specifically for checking monitor specifications against blockchain transactions. Currently we are doing it in _offline mode_ by first downloading transactions using `solarcraft fetch`, and then verifying them; eventually we want to move into verifying monitor specs on the live blockchain, i.e. we want to do _online monitoring_.

If you are still interested in the details -- continue reading!



## Blockchain Runtime Monitors in Formal Attire 👔

In this section we define, using mathematical notation, what blockchain monitors are, and how to verify whether a blockchain transaction satisfies the conditions expressed by a monitor.

Formally, a blockchain is a sequence of _ledgers_, where each ledger is a snapshot of the blockchain _environment_ and the blockchain _state_. States are partitioned: first into separate spaces per contract, and then into separate regions per contract variable. Blockchain states are mutated by _transactions_, where each transaction is an invocation of a certain contract _method_ with the corresponding method parameters supplied. The invoked method modifies the states according to its logic. A successful transaction bring the blockchain from one environment/state to the next; a rejected/reverted transaction leaves the blockchain environment/state unchanged. We assume that unsuccessful transactions can be still observed.

We employ the following notation:

- $$D$$ is the set of all possible data values: strings, numbers, structs, etc. Mathematically we don't distinguish between different data types (though practically we of course do).
- $$V$$ is the set of typed contract variables. At this stage we don't distinguish between states of different contracts: logical assertions may refer to the state of any contract (e.g. to token balances in other contracts).
- $$S = S_0, S_1, ...$$ is the sequence of states.
- $$S_i \subseteq V \rightarrow D$$ is the $$i$$-th contract state, which is a partial mapping from variables to their data values. If a variable $$v$$ is present in the mapping $$S_i$$, we say that it _exists_ in this state.
- $$T = T_0, T_1, ...$$ is the sequence of transactions. Each transaction brings the contract into its next state, which we denote by $$S_i \xrightarrow{T_i} S_{i+1}$$.
- $$P_T$$ is the set of all possible typed method parameters.
- $$T_i \subseteq P_T \rightarrow D$$: each transaction is a method invocation, represented by a partial mapping from method parameter names to their values; only the parameters specific to the invoked method are present in the mapping.
- $$E = E_0, E_1, ...$$ is the blockchain environment, which is a sequence of environment states; each transaction executes in a specific environment state.
- $$P_E$$ is the set of all typed environment parameters (such as `current_contract_address`, `ledger_timestamp`, or `method_name`).
- $$E_i = P_E \rightarrow D$$ is a mapping from environment parameters to their values, and defines the current blockchain environment, in which $$T_i$$ executes.
- $$X_i \in \mathbb{B} = \{ \top, \bot \}$$ is the result of executing the transaction $$T_i$$: $$\top$$ in case of success, $$\bot$$ in case of failure.

The above definitions describe the structure of the object to which we apply monitor specifications: a smart contract, executing on a blockchain. Now it's time to define the structure of monitor specifications themselves. As checking each direct method specification or reverse effect specification is independent from others, we define only the structure for individual monitors.

- $$M_D = \langle F, P, H \rangle$$ is a direct method monitor specification, where the components are the finite sets of `MustFail`, `MustPass`, and `MustHold` conditions respectively.
- $$M_R = \langle C, A \rangle$$ is a reverse effect monitor specification, where the components are the finite sets of `MonitorCheck` and `MonitorAssert` conditions respectively.

In the above:

- For any $$F_j \in F$$, $$P_k \in P$$ we have $$F_j, P_k \subseteq (E_i, S_i, T_i) \rightarrow \mathbb{B}$$ are boolean conditions of the environment state, the past contract state, and the method parameters.
-  For any $$H_j \in H$$ we have $$H_j \subseteq (E_i, S_i, T_i) \times S_{i+1} \rightarrow \mathbb{B}$$ are boolean conditions of the environment state, the past contract state, the method parameters, and the next contract state.
- For any $$C_j \in C$$, $$A_k \in A$$ we have $$C_j, A_k \subseteq (E_i, S_i) \times S_{i+1} \rightarrow \mathbb{B}$$ are boolean conditions defined over the environment state, the past contract state, and the next contract state.

### Verification Conditions for Blockchain Monitors

_Verification conditions_ are verifiable mathematical statements, which encode a certain aspect of the system correctness; in our case they encode whether the blockchain transaction is correct wrt. the blockchain monitor. Having formally defined what are blockchain states, transactions, and monitors, we are now in a position to specify monitor verification conditions.

**For a direct blockchain monitor** $$M_D = \langle F, P, H \rangle$$, we combine individual monitor conditions into larger ones:

$$\mathbb{C}_{\mathit{Fail}} = \bigvee_{j}{F_j}$$

$$\mathbb{C}_{\mathit{Pass}} = \bigvee_{j}{P_j}$$

$$\mathbb{C}_{\mathit{Hold}} = \bigwedge_{j}{H_j}$$

Given the above combined conditions, we check these verification conditions:

| Name | Verification condition |
| -----| ---------------------- |
| Must fail | $$\mathbb{C}_{\mathit{Fail}}  \implies (X_i = \bot)$$ |
| Failure completeness | $$(X_i = \bot) \implies \mathbb{C}_{\mathit{Fail}}$$ |
| Must succeed | $$\neg \mathbb{C}_{\mathit{Fail}} \wedge \mathbb{C}_{\mathit{Pass}} \implies (X_i = \top)$$ |
| Success completeness | $$(X_i = \top) \implies \neg \mathbb{C}_{\mathit{Fail}} \wedge \mathbb{C}_{\mathit{Pass}}$$ |
| Method correctness  | $$(X_i = \top) \implies \mathbb{C}_{\mathit{Hold}}$$ |

Please feel free to contrast these formal verification conditions with the [informal conditions from the previous post][part4directmonitors], as well as with the [TLA+ encoding of verification conditions for Timelock's `deposit` method][depositVCs]. Notice also that the two implications from the pairs "Must fail"/"Failure completeness" and "Must succeed"/"Success completeness" encode together an equivalence between the checks and the transaction execution result. Nevertheless, we consider it a better strategy to treat these conditions separately, as this allows the developers to encode a more fine-grained monitor response. For example, a monitor may forcefully revert a transaction that violates the "Must fail" condition, but only issue a warning when "Failure completeness" is violated.

**For a reverse blockchain monitor** $$M_R = \langle C, A \rangle$$, we also combine individual monitor conditions into larger ones:

$$\mathbb{C}_{\mathit{Check}} = \bigvee_{j}{C_j}$$

$$\mathbb{C}_{\mathit{Assert}} = \bigwedge_{j}{A_j}$$

Reverse monitors encode only a single verification condition:

| Name | Verification condition |
| -----| ---------------------- |
| Effect correctness | $$(X_i = \top) \wedge \mathbb{C}_{\mathit{Check}} \implies \mathbb{C}_{\mathit{Assert}}$$ |

You may compare the above verification condition with the [informal condition from the previous post][part4reversemonitors], as well as with the [TLA+ encoding of verification conditions for `BalanceRecord` monitor][balanceRecordVCs].


### Model Checking Blockchain Monitors

_Model checking_ is an automatic procedure of verifying mathematical specifications. Within [Solarkraft][], we employ [TLA+][] as our specification language, and [Apalache][] as our model checker. Here are a few details worth noting:

- Apalache is a _general purpose_ model checker, in that it performs _invariant checking_: given an initial system state $$\mathit{Init}$$, an encoding of the system transitions (the _next-state relation_) $$\mathit{Next}$$, and an encoding of a supposed system invariant $$\mathit{Inv}$$, it checks whether the invariant does indeed hold in all system states reachable from the initial one by executing system transitions.
- Apalache is a _bounded_ model checker: it can check invariants only in states reachable in a certain number of transition steps (the execution bound $$\mathit{Length}$$, say 1, 5, or 10) from the initial state.
- Apalache is a _symbolic_ model checker, i.e. it encodes the the verification conditions symbolically, as formulas in certain logical theories, and passes the resulting encoding to _Satisfiability Modulo Theories (SMT) solvers_, which are specialized tools for solving massive volumes of math equations.

We employ Apalache by encoding monitor verification conditions as an invariant checking problem. For any given blockchain environment $$E_i$$, the transaction pre-state $$S_i$$, the transaction being executed $$T_i$$, the transaction execution result $$X_i$$, the transaction post-state $$S_{i+1}$$, as well as any of the above verification conditions $$\mathit{VC}$$, we execute Apalache using the following encoding:

- Initial state: $$\mathit{Init} = E_i \wedge S_i$$
- Next-state relation: $$\mathit{Next} = T_i \wedge S_{i+1}$$
- Invariant: $$\mathit{Inv} = \mathit{VC}$$
- Execution bound: $$\mathit{Length} = 1$$


A few TLA+ tests for Apalache verification conditions can be found e.g. in [deposit_test.tla](https://github.com/freespek/solarkraft/blob/cf26a544ab204220eab62a3545300cb689aa899b/doc/case-studies/timelock/deposit_test.tla) (for the direct monitor of Timelock's `deposit` method), or in [balance_record_test.tla](https://github.com/freespek/solarkraft/blob/cf26a544ab204220eab62a3545300cb689aa899b/doc/case-studies/timelock/balance_record_test.tla) (for the reverse balance record monitor). In all cases Apalache is invoked in a similar fashion, e.g. like that for `deposit`'s first test:

```sh
apalache-mc check --length=1 --init=Init_1 --next=Next_1 deposit_test.tla
```

As explained above, Apalache is a _bounded_ model checker, but, specifically for monitoring, this restriction is irrelevant: Apalache is blazing fast at checking the above formulas!

One important point of consideration is the flexibility of encoding monitor verification conditions for model checking. While in the above tests a monolithic encoding is used, they can also be encoded fine-grained, down to the smallest scale of a single direct monitor condition (one of $$F_j$$, $$P_j$$, $$H_j$$), or of a single reverse monitor condition (one of $$C_j$$, $$A_j$$). While for the Timelock example it doesn't make any difference wrt. the Apalache execution speed (it's very fast both at the finest and at the coarsest level), for other monitors the difference may be substantial; we will return to this point later on.

### Practical checking of monitor specifications

In the present [Solarkraft system][Solarkraft] we do what's called _offline monitoring_: we verify monitors _after_ the state has already been committed to the blockchain. Our eventual goal is to perform _online monitoring_, i.e. to verify the monitors _before_ the state has been committed, in order to be able to do preventive actions. This far-reaching goal is non-trivial, and has a few intermediate-strength solutions, which we are about to explore now.

The 

### Offline monitoring




_Development of Solarkraft was supported by the [Stellar Development Foundation][] with a generous Activation Award from the [Stellar Community Fund][] of 50,000 USD in XLM._


[Solarkraft]: https://github.com/freespek/solarkraft
[Apalache]: https://konnov.phd/portfolio/apalache/
[part1]: https://thpani.net/2024/06/why-smart-contract-bugs-matter-and-how-runtime-monitoring-saves-the-day-solarkraft-1/
[part2]: https://thpani.net/2024/06/small-and-modular-runtime-monitors-in-tla-for-soroban-smart-contracts-solarkraft-2/
[part3]: https://protocols-made-fun.com/solarkraft/2024/06/19/solarkraft-part3.html
[part4]: https://systems-made-simple.dev/solarkraft/2024/06/24/solarkraft-hybrid-monitors.html
[part4directmonitors]: https://systems-made-simple.dev/solarkraft/2024/06/24/solarkraft-hybrid-monitors.html#direct-monitors
[part4reversemonitors]: https://systems-made-simple.dev/solarkraft/2024/06/24/solarkraft-hybrid-monitors.html#reverse-monitors
[depositVCs]: https://github.com/freespek/solarkraft/blob/cf26a544ab204220eab62a3545300cb689aa899b/doc/case-studies/timelock/deposit.tla#L66-L104
[balanceRecordVCs]: https://github.com/freespek/solarkraft/blob/cf26a544ab204220eab62a3545300cb689aa899b/doc/case-studies/timelock/balance_record.tla#L31-L39
[Igor Konnov]: https://konnov.phd
[Jure Kukovec]: https://www.linkedin.com/in/jure-kukovec/
[Andrey Kuprianov]: https://www.linkedin.com/in/andrey-kuprianov/
[Thomas Pani]: https://thpani.net

[Soroban]: https://stellar.org/soroban
[Stellar Community Fund]: https://communityfund.stellar.org
[Stellar Development Foundation]: https://stellar.org/foundation

[Stellar]: https://en.wikipedia.org/wiki/Stellar_\(payment_network\)
[TLA+]: https://en.wikipedia.org/wiki/TLA%2B

[transaction]: https://developers.stellar.org/docs/learn/fundamentals/transactions/transaction-lifecycle
