# Contents

* [Purpose](#purpose)
* [Background](#background)
  * [What is Coin Selection?](#what-is-coin-selection)
  * [Why is Coin Selection Non-Trivial?](#why-is-coin-selection-non-trivial)
* [Common Interface](#common-interface)
  * [Parameters](#parameters)
  * [Results](#results)
  * [Failure Modes](#failure-modes)
* [Implementations](#implementations)
  * [Largest-First](#largest-first)
  * [Random-Improve](#random-improve)
    * [Motivating Principles](#motivating-principles)
      * [Principle 1: Dust Management](#principle-1-dust-management)
      * [Principle 2: Change Management](#principle-2-change-management)
      * [Principle 3: Performance Management](#principle-3-performance-management)
    * [Phases of Computation](#phases-of-computation)
      * [Phase 1: Random Selection](#phase-1-random-selection)
      * [Phase 2: Improvement](#phase-2-improvement)
    * [Termination](#termination)

# Purpose

The purpose of this article is to describe, in human-readable terms, the coin
selection algorithms used by Cardano Wallet and other parts of the Cardano
ecosystem.

# Background

## What is Coin Selection?

Coin selection is the process of choosing unspent coins from a wallet in order
to pay money to one or more recipients.

Similarly to how a physical wallet holds value in the form of unspent coins and
banknotes, a Cardano wallet holds value in the form of _unspent transaction
outputs_ (UTxO), which form a set. Each output in the UTxO set has a particular
_value_, and the total value of a wallet is the _sum_ of these values.

When making a payment with money held in a physical wallet, we typically select
a number of coins and banknotes from the wallet that, when added together,
_cover_ the required amount. Ideally, we'd always be able to select _just
enough_ to cover the exact amount required.  However, given that coins and
banknotes have fixed values (and cannot be subdivided), it's often _impossible_
to select the exact amount required. In such cases, we typically give the
recipient _more_ than the required amount, and then receive the excess value
back as _change_.

Similarly, when using a Cardano wallet to make a payment, the wallet software
must choose unspent outputs from the wallet's UTxO set, so that the total value
of selected outputs is enough to cover the target amount. Just as with physical
money, outputs from the UTxO cannot be subdivided, and must be spent completely
in a given transaction. Consequently, in the case where it's not possible to
cover the target amount exactly, the wallet software must arrange that change
is paid back to the wallet.

Coin selection refers to the process of choosing unspent outputs from a
wallet's UTxO set in order to make a payment, and arranging for change to be
paid back to the wallet by creating one or more change outputs.

## Why is Coin Selection Non-Trivial?

There are a number of issues which make the problem of coin selection more
complicated than it would initially appear.

 * Each transaction has a maximum size, as defined by the protocol. The size of
   a transaction increases as we add more inputs or outputs.

   Therefore, there's a practical limit on the number of coins we can select
   for any given transaction.

 * The most obvious strategy for coin selection, which consists of trying to
   get as close to the requested value as possible, will tend (over time) to
   create a lot of dust: small unspent outputs.

   Dust outputs are a problem, because even if the total value of dust in a
   wallet is more than enough to cover a given target amount, if we attempt to
   include that dust in a given transaction, we may run out of space (by
   reaching the transaction size limit) before we can cover the target amount.

 * The most obvious strategy for paying change, which consists of making a
   single change output with the exact excess value, will tend (over time) to
   reduce the size of the UTxO set. This is bad for two reasons.

   Firstly, having a small UTxO set limits the number of payments that we can
   make in parallel. Since a single UTxO entry can only be used to pay for a
   single output, we need at least as many UTxO entries as there are outputs.

   Secondly, the approach of coalescing all change into a single output is
   considered to be bad for privacy reasons.

There are several properties that we would like a coin selection algorithm
to have:

 * A coin selection algorithm should, over the course of time, aim to generate
   and maintain a UTxO set with "useful" outputs; that is, outputs that allow
   us to process future payments with a minimum number of inputs.

 * A coin selection algorithm should employ strategies to limit the
   amount of dust that accumulates in the UTxO set.

# Common Interface

In general, algorithms used for coin selection implement a common interface.

This section will describe the parameters that are common to all coin selection
algorithms described by this article, the results that they produce, and the
failure modes that they have in common.

## Parameters

### _Initial UTxO Set_

This corresponds to the UTxO set of a wallet.

### _Requested Output List_

A list of requested outputs, where each output is a 2-tuple that consists of a
_target address_ and an _amount_.

### _Maximum Input Count Function_

## Results

### _Final UTxO Set_

### _Final Coin Selection_

## Failure Modes

There are a number of ways in which a coin selection algorithm can fail.

This section categorizes the various failure modes.

### UTxO Balance Insufficient

This failure occurs when the total value of the entries within the [initial
UTxO set](#initial-utxo-set) (the amount of money _available_) is _less than_
the the total value of all entries in the [initial output
list](#requested-output-list) (the amount of money _required_).

### UTxO Not Fragmented Enough

This failure occurs when the _number_ of entries in the [initial UTxO
set](#initial-utxo-set) is _smaller than_ the number of entries in the
[requested output list](#requested-output-list).

All algorithms require that there is _at least one_ UTxO entry available _for
each_ output.

### UTxO Fully Depleted

This failure occurs if the algorithm depletes all entries from the [initial
UTxO set](#initial-utxo-set) _before_ it is able to pay for all outputs in the
[requested output list](#requested output list).

### Maximum Input Count Exceeded

This failure occurs if the _number_ of UTxO entries needed to pay for the
outputs in the [requested output list](#requested-output-list) exceeds the
upper limit specified by the [maximum input count
function](#maximum-input-count-function).

# Implementations

## Largest-First

## Random-Improve

The **Random-Improve** coin selection algorithm works in **two phases**.

In the first phase, the algorithm iterates through each of the given outputs in
descending order of coin value. For each output, the algorithm randomly selects
UTxO entries until the total value of selected entries is enough to pay for the
ouput.

In the second phase, the algorithm attempts to improve upon each of the UTxO
selections made in the previous phase, by conservatively expanding the
selection made for each output, in order to generate improved change values.

### Motivating Principles

There are several motivating principles behind the design of the algorithm.

#### Principle 1: Dust Management

The probability that random selection will choose dust entries from a UTxO
set _increases_ with the proportion of dust in the set.

Therefore, for a UTxO set with a large amount of dust, there's a high
probability that a random subset will include a large amount of dust.

Over time, selecting entries randomly in this way will tend to _limit_ the
amount of dust that accumulates in the UTxO set.

#### Principle 2: Change Management

As mentioned in the [background](#background) section, it is desirable that
coin selection algorithms, over time, are able to create UTxO sets that have
_useful_ outputs: outputs that will allow us to process future payments with a
minimum number of inputs.

If for each payment request of value **v** we create a change output of
_roughly_ the same value **v**, then we will end up with a distribution of
change values that matches the typical value distribution of payment
requests.

#### Principle 3: Performance Management

Searching the UTxO set for additional entries to _improve_ our change outputs
is _only_ useful if the UTxO set contains entries that are sufficiently
small enough. But it is precisely when the UTxO set contains many small
entries that it is less likely for a randomly-chosen UTxO entry to push the
total above the upper bound.

### Phases of Computation

This section will describe the phases of the algorithm in detail.

#### Phase 1: Random Selection

In this phase, the algorithm iterates through each of the given outputs. For
each output, the algorithm randomly selects UTxO entries until the total value
of selected entries is enough to pay for the ouput.

During this phase, the algorithm:

  *  processes outputs in /descending order of coin value/.

  *  maintains a /remaining UTxO set/, initially equal to the given
     /UTxO set/ parameter.

For each output of value __/v/__, the algorithm /randomly/ selects entries
from the /remaining UTxO set/, until the total value of selected entries is
greater than or equal to __/v/__. The selected entries are then associated
with that output, and removed from the /remaining UTxO set/.

This phase ends when every output has been associated with a selection of
UTxO entries.

However, if the remaining UTxO set is completely exhausted before all
outputs can be processed, the algorithm terminates and falls back to the
__Largest-First__ algorithm.

#### Phase 2: Improvement

In this phase, the algorithm attempts to improve upon each of the UTxO
selections made in the previous phase, by conservatively expanding the
selection made for each output in order to generate improved change
values.

During this phase, the algorithm:

  *  processes outputs in /ascending order of coin value/.

  *  continues to maintain the /remaining UTxO set/ produced by the previous
     phase.

  *  maintains an /accumulated coin selection/, which is initially /empty/.

For each output of value __/v/__, the algorithm:

 1.  __Calculates a /target range/__ for the total value of inputs used to
     pay for that output, defined by the triplet:

     (/minimum/, /ideal/, /maximum/) = (/v/, /2v/, /3v/)

 2.  __Attempts to /improve/ upon the /existing UTxO selection/__ for that
     output, by repeatedly selecting additional entries at random from the
     /remaining UTxO set/, stopping when the selection can be improved upon
     no further.

     A selection with value /v1/ is considered to be an /improvement/ over a
     selection with value /v0/ if __all__ of the following conditions are
     satisfied:

      * __Condition 1__: we have moved closer to the /ideal/ value:

            abs (/ideal/ − /v1/) < abs (/ideal/ − /v0/)

      * __Condition 2__: we have not exceeded the /maximum/ value:

            /v1/ ≤ /maximum/

      * __Condition 3__: when counting cumulatively across all outputs
      considered so far, we have not selected more than the /maximum/ number
      of UTxO entries specified by 'maximumInputCount'.

 3.  __Creates a /change value/__ for the output, equal to the total value
     of the /final UTxO selection/ for that output minus the value /v/ of
     that output.

 4.  __Updates the /accumulated coin selection/__:

      * Adds the /output/ to 'outputs'.
      * Adds the /improved UTxO selection/ to 'inputs'.
      * Adds the /change value/ to 'change'.

This phase ends when every output has been processed, __or__ when the
/remaining UTxO set/ has been exhausted, whichever occurs sooner.

### Termination

When both phases are complete, the algorithm terminates.

The /accumulated coin selection/ and /remaining UTxO set/ are returned to
the caller.

