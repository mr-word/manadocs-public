## Intro

Let us start by saying that we are making a serious effort to implement the primitives we need directly in Bitcoin L1.
Indeed, the "`OP_PUSH_TX` revolution" began when we coaxed nChain into showing us how to do this trick by
[asking them if there was something we are missing](https://github.com/bitcoin-sv-specs/protocol/issues/5). We were briefly very optimistic that this would allow us to do
everything we need, but we are now at a stage where we are encountering another serious limitation.
We are uncompromising in our requirements for L1 UTXO functionality.
We hope to learn that there is (or will be) a solution that meets our needs directly in L1, so that we do not need to create an overlay UTXO system.

First we will describe the problem and review existing solutions.

Then we will propose a UTXO overlay network with a small change that solves the problem while **preserving the key property of UTXO validation, which is that it is maximally parallel**.

Finally we will discuss how Bitcoin could support this functionality with only a soft fork (assuming a hard fork to support our extended UTXO type is off the table).

It is impossible to describe the entire range of features we are about to describe using a finite set of examples,
just as it is not possible to describe every kind of script that could be written with a scripting language.
Still, it is helpful to use a concrete example.

## Review of existing approaches

Let's use a simple example: making an NFT whose holder is able to spend special UTXOs that "belong to" that NFT.
You can imagine easily extending this into a name registry / user ID system.
Later we will show how a single unique ID system is sufficient to enable the general case, and is the 'missing link' required to fully emulate EVM-like functionality in pure UTXO,
whose power comes from *stateful objects that can atomically communicate without a-priori knowledge of each other*.

Let's review the general types of approaches, up to the current most sophisticated pattern which is [xhliu tokens](https://archive.is/uCWQh).
In all cases except xhliu tokens, oracles are required to attest to the validity of the user-defined colored UTXO *in L1 -- that is, when read by other UTXOs*.
In the last case, the user-defined 'coloring' can be validated entirely in L1 (by other UTXOs, using `OP_PUSH_TX`),
but there is a critical limitation in that the transaction size increases with each use (the entire history is in the transaction).

(We ask the reader not to jump to "solutions" that only address the particulars of this application.
Use some imagination to think of how a global "rich state" could be utilized. Hint: think of it as a computer's memory address space.
Imagine a transaction as a transient "lock", and UTXOs as intermediate state snapshots / continuations. Crucially, imagine doing this *without UTXOs that depend on oracles*)

* Things like Moneybutton IDs or Twetch IDs use a completely off-chain database.
For a UTXO to be aware of the current name->key mapping, these services would have to act like an oracle,
and the UTXO would have to be aware of the key rotation strategy and other details of the oracle service.
* Another example of the "BSV as just a data bus" is the [Magic Attribute Protocol](https://map.sv/).
This is not an NFT protocol but is worth highlighting here to help explain the point.
You can imagine a script that does some complex logic depending on the value of the MAP database -- except that a UTXO would
need to be fed those values by an oracle.
* "Colored coin" approaches utilize the L1 script language for spend conditions, but determining
if a given UTXO is a valid "colored" UTXO requires validating the entire colored UTXO history. This means
that a UTXO that wants to depend on such a colored UTXO would require an oracle to attest to its validity.
* [Allegory](https://www.xoken.org/technology/allegory-allpay-protocol-suite/) is a particularly interesting "colored UTXO" naming solution,
because it is well-suited to being enforced by miners with a soft fork. More on this later...
* Tokens utilizing `OP_PUSH_TX` to enforce forward conditions. This is an improvement over regular colored
UTXO in the sense that they cannot accidentally be spent incorrectly in a way that is invalid according
to the 'colored' logic. Unfortunately, due to problems described in xhliu's post on xhliu tokens, it is
still not possible to be sure that a given UTXO is a valid colored UTXO without validating the entire
colored subset of the transaction history, and so it would have to be fed by an oracle to any UTXO that wishes to depend on this state.

* Finally, [Xhliu Tokens](https://archive.is/uCWQh) provide a solution that *is* verifiable from another UTXO script. A UTXO script can
have logic that depends on an xhliu token in another UTXO, and verify this logic entirely in-script.
Unfortunately, this comes with a different critical constraint: The entire xhliu token transaction history is contained in the xhliu UTXO.
Each "use" (read or state change) of that "object" increases it's size. This quickly becomes intractable when composing distinct L1 apps using this strategy.

`OP_PUSH_TX` gives us what we call "local statefulness". But the UTXO architecture could support *global* statefulness, if only we could depend on even a single unique ID system in L1.

## manaflow and unique refs in UTXOs
Our solution: A [distinct UTXO architecture](https://github.com/mr-word/manaflow-poc3) which enforces globally unique
references in a new `uref` field on a UTXO, using only local (per-transaction) validation rules.
This is essentially solving the "first issuance" problem described in xhliu using native L1 (although "issuance" is too specific of a word, since the script could contain any logic).
No additional indices besides the existing UTXO set are required, and most crucially, this approach does not change the maximally parallel nature of UTXO-style transaction validation.
The rules are simple and local, but result in a globally valid unique ID space.

If a uref appears in an output to a transaction, then either
1) The uref must appear in at least one input, or
2) The uref must be the hash of (txid,index) of one of the inputs.

This hash is very similar in spirit to the issuance transaction hash in xhliu tokens.
This extra rule only enforces when a uref can *first* appear. It is up to the script to impose additional constraints in forward conditions, which are app-dependent.
In our case, only one output can contain the special NFT UTXO.

The key functionality enabled here is that now *other UTXOs* can make reference to the uref hash, and can be certain that all UTXOs with this uref were subject
to the initial forwarding conditions.

## how to do it in bitcoin L1 with a hard fork
Arguably the principled use of the "version" byte in each transaction does not contradict the rhetoric on "set in stone"...
We just need *any* globally-unique ID system that can be forwarded through UTXOs,
but we also introduce a number of ergonomic improvements in our architecture that could also be added.
Again, we wish to emphasize that all these 'ergonomic' improvements could be compiled down to bitcoin L1 with appropriate use of unrolling, inlining, and introspection (`OP_PUSH_TX`).
But unique references cannot be compiled down because they are a feature of transaction-level validation, not script-level validation.

## how to do it in bitcoin L1 with just a soft fork
If miner enforce even a single "unique ref" that can be forwarded through UTXOs, it would be sufficient to use
as a building block to create any number of L1 apps.
One simple implementation could be to treat script prefix `'ID' <64-bit identifier> OP_2DROP` (which is effectively a NOP) like our 'uref', and distribute these to miners.
Allegory is another good candidate for soft-fork enforcement.
