# Purpose

The purpose of this article is to describe, in human-readable terms, the coin
selection algorithms used by Cardano Wallet and other parts of the Cardano
ecosystem.

# What is Coin Selection?

Coin selection is the process of choosing unspent coins from a wallet in order
to pay money to one or more recipients.

Similarly to how a physical wallet holds value in the form of unspent coins and
banknotes, a Cardano wallet holds value in the form of unspent transaction
outputs (UTxO), which form a set. Each output in the UTxO set has a particular
value, and the total value of a wallet is the sum of these values.

When making a payment with money held in a physical wallet, we must typically
select a number of coins and banknotes to give to the recipient. Ideally, we'd
always be able to select just enough to cover the exact amount required.
However, given that coins and banknotes have fixed values (and cannot be cut in
half), it's often impossible to select the exact amount required. In such
cases, we typically give the recipient more than the required amount, and then
receive the excess value back as change.

Similarly, when using a Cardano wallet to make a payment, the wallet software
must choose unspent outputs from the wallet's UTxO set, so that the total value
of selected outputs is enough to cover the target amount. Just as with physical
money, outputs from the UTxO cannot be subdivided, and must be spent completely
in a given transaction. In the case where it's not possible to cover the target
amount exactly, the wallet software must arrange that change is paid back to
the wallet.

Coin selection refers to the process of choosing unspent outputs from a
wallet's UTxO set in order to make a payment, and arranging for change to be
paid back to the wallet appropriately.

# Why is Coin Selection Non-Trivial?

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

   Secondly, the approach of coalescing all change into a single output may
   have poor privacy characteristics.

# What Properties of a Coin Selection Algorithm are Desirable?

Ideally, coin selection algorithms should, over time, create a UTxO set that
has "useful" outputs; that is, outputs that allow us to process future payments
with a minimum number of inputs.
