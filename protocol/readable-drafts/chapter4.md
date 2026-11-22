# 4 Abstract Protocol

## 4.1 Cryptographic tools

A public ledger lets every node check every payment by showing everything, and a shielded payment has to hide the same facts without losing the check. Zcash manages the trade with a small set of cryptographic tools, each described here by the guarantee the tool gives rather than by the algorithm behind the guarantee.

**Commitments.** A note has to go onto the block chain without being readable, and a commitment works like a sealed envelope, hiding the note while fixing the contents so nobody can change them later.

**Pseudo-random functions.** A spent note then needs a mark only the owner can compute and nobody can connect to the envelope. A pseudo-random function, a keyed hash whose outputs look random to anyone without the key, gives exactly such a mark, and a note's nullifier is one of these outputs under a key belonging to the owner.

**Zero-knowledge proofs.** The owner must still convince every node the hidden note exists and belongs to the owner. A zero-knowledge proof shows secret values pass a public list of checks while revealing nothing else about the values, and a correct proof always verifies, while nobody can feasibly prove a false claim. Sprout proofs use BCTV14 and later Groth16, Sapling proofs use Groth16, and Orchard and Ironwood proofs use Halo 2.

**Signatures.** A proof on the block chain could be copied into another transaction, or altered by someone without the secret values into a proof about related values, and signatures over the whole transaction prevent both. Zcash uses two unusual kinds of signature, one whose key can be re-randomized for each spend and one whose keys can be added together.

**Value commitments.** Amounts are hidden too, yet a node must still check the value going into a transaction matches the value coming out. The value commitments of Sapling and Orchard can be added together without being opened, and a signature whose keys add in the same way shows the hidden amounts balance.

**Encryption.** The recipient has to learn what the note contains while nobody else does. Key agreement gives the sender and the recipient one shared secret, a key derivation function turns the secret into a key, and authenticated encryption seals the note, hiding both the contents and the address the note is for.

**Hash functions.** Nobody can feasibly find two inputs to a hash function with the same output, and hash functions build the tree of note commitments and give the fixed curve points behind commitments and addresses.

## 4.2 Keys

A wallet with one key for everything would put the power to spend at risk every time the wallet only needed to read or receive a payment. Zcash instead derives a chain of keys from one random spending key, and each step runs one way, so a weaker key can be handed out without exposing the keys above.

**Sapling.** The spending key gives three secrets, the spend authorizing key for approving spends, the nullifier private key for computing nullifiers and the outgoing viewing key for reading payments sent. The public halves *ak* and *nk* of the first two, together with the outgoing viewing key, form the full viewing key, and a hash of *ak* and *nk* gives the incoming viewing key, which finds and reads payments received.

**Orchard and Ironwood.** The keys keep the same roles, but the outgoing viewing key and a diversifier key come from the full viewing key rather than straight from the spending key. Wallets must derive Orchard keys exactly as specified, or the notes may not be recoverable through the recovery protocol of ZIP 2005 if attacks with quantum computers become practical.

**Sprout.** The spending key gives one paying key and one receiving key, and each spending key has exactly one address.

**Addresses.** A Sapling or Orchard address pairs a diversifier with a transmission key made from the diversifier and the incoming viewing key, so every new diversifier gives a new address. The addresses work like post-office boxes forwarding to one mailroom, where outsiders cannot tell which boxes belong together while the one incoming viewing key finds the mail sent to every box. Orchard makes each diversifier from a diversifier index, and wallets should not pick the index at random (ZIP 32).

**Failed derivations.** A derivation can now and then produce a value which is not a valid key, and the wallet then discards the spending key and starts again with a new one.

## 4.3 Shielded parts of a transaction

Nodes cannot see a shielded transfer, so the transaction carries a description of the transfer instead, holding the public values the checks need and a proof the hidden values behind them are right. No description shows a note or a shielded address, every field must be a valid encoding, and every proof must verify.

**JoinSplits.** A Sprout JoinSplit spends two notes and creates two, and two public amounts move value in or out of the transparent side of the transaction, at most one of them non-zero. Since Canopy the amount moving into Sprout must be zero, which closes the Sprout pool to new value.

**Spends and Outputs.** Sapling gives each spent note a Spend, which carries a spend authorization signature over the whole transaction, and each new note an Output, which carries the encrypted note and no signature. The value commitment and the randomized key of a Spend, and the value commitment and the ephemeral key of an Output, must not be points of small order.

**Actions.** Orchard and Ironwood join one optional spend and one optional output into each Action, with one value commitment for the net value. Two flags for each pool say whether the Actions may spend and may create non-zero value, and from NU6.3 a third flag says whether a new note may go to a different address from the spent note. Every field of an Action must be a canonical encoding, and the randomized key must not be the identity point. The proof must verify under the verifying key of the circuit in force, and the circuit changed at NU6.2, to fix a flaw which could have let a transaction break the balance rule, and again at NU6.3.

## 4.4 Sending notes and dummy notes

A sender has the recipient's address and no private channel to the recipient, so the sender builds the whole note alone and puts everything the recipient needs on the block chain. The sender chooses fresh random values, computes the value commitment and the note commitment, encrypts the note to the recipient and makes the proof.

Since Canopy most of a note's random values come from one 32-byte seed, *rseed*, which travels inside the encrypted note so the recipient can rebuild the note exactly. In Orchard the new note's *ρ* is the nullifier of the note spent in the same Action. The sender also adds a second, shorter ciphertext readable by an outgoing viewing key, which lets the sender read the payment again later, or readable by no key at all, which keeps the payment secret even if the sender's keys later leak.

Wallets should shuffle the order of spends and outputs to reveal less about the transaction. Since Canopy wallets must not send to Sprout addresses (ZIP 211).

**Dummy notes.** A transaction can still give away how many notes the sender really spends, and dummy notes hide the count. A dummy note is a note of zero value under a throwaway key, like a cheque made out for nothing, which fills a place in a transaction without moving money. JoinSplits use dummies to fill the fixed room for two notes in and two out, and Sapling and Orchard use dummy spends to hide the number of real spends. An Orchard Action with no real note to spend should spend a dummy note and take the dummy's nullifier as the new note's *ρ*. A dummy passes every check a real note passes except one, as the proof does not require a note of zero value to be in the tree.

## 4.5 Merkle paths

A spender has to show the spent note is one of the notes on the block chain without saying which one. Each pool keeps the pool's note commitments in a Merkle tree of fixed depth which only grows. A Merkle path lists the neighbouring hash at each level from one note commitment up to the root, so anyone holding the commitment and the path can recompute the root. The spender keeps the commitment and the path secret inside the proof, and the transaction names only the root, called the anchor, so nodes learn the note is in the tree but not where.

## 4.6 Signatures and balance

Once the parts are built, nobody should be able to change the transaction, and every node must be able to check the hidden amounts balance. Each signature in a shielded transaction signs the SIGHASH, a hash of the whole transaction, computed for version 4 as ZIP 243 specifies, for version 5 as ZIP 244 specifies and for version 6 as ZIP 229 specifies. From NU5 the SIGHASH type must be one of the defined types in canonical form, and every transaction must name the consensus branch of the upgrade in force.

**Sprout.** Sprout signs each transaction with a throwaway key used once, and the proof ties the key to the owners of the spent notes through the value *h_Sig*. Sprout also checks the balance inside the proof.

**Balance.** Sapling and Orchard check the balance outside the proof, with value commitments which can be added together. A commitment to a value *v* with a random trapdoor *rcv* is

*cv* = *v*·V + *rcv*·R

for two fixed points V and R, so adding commitments adds the values and the trapdoors separately. A node adds the commitments of the spends, subtracts those of the outputs and subtracts the public amount the pool releases to the transparent side. In an honest transaction the values cancel and only a multiple of R remains, a public key whose private key, the sum of the trapdoors, only the builder of the transaction knows.

The builder signs the transaction with the private key, and the signature, called the binding signature, must verify under the public key each node computes from the commitments. If the values did not cancel, a valid signature would need one commitment opened to two different values, which the commitments rule out. Every value fits in 64 bits and a transaction holds only so many parts, so the sum cannot wrap round to zero either. Proving knowledge of the trapdoors also stops anyone copying an Output into another transaction, and Orchard and Ironwood work the same way, with one net value commitment per Action and one binding signature per pool.

**Spend authorization.** The binding signature shows the values balance but not the spender's right to spend them, and a separate spend authorization signature proves the spender holds the key of each spent note. Keeping the signature outside the proof lets a device with little memory or computing power, such as a hardware wallet, approve a spend without building the proof. A signature checked against the owner's own key would link every spend to the owner's address. The wallet instead shifts the key by a fresh random value for each spend and publishes the shifted key *rk*, the owner's key in a fresh disguise, and the proof shows *rk* disguises the right key. A wallet may hand the proof to a helper, who then learns enough to see the owner's payments but still cannot spend.

## 4.7 Nullifiers and pool balances

A nullifier has to follow from the note alone, differ from the nullifier of every other note, and stay unlinkable to the note for anyone without the owner's key, even the sender. Each note therefore has a value *ρ* no other note shares, which Sprout and Orchard notes carry inside the note and Sapling computes from the note commitment and the note's place in the tree. The nullifier is computed from *ρ* under a key of the owner, the spending key in Sprout and the nullifier deriving key in Sapling and Orchard.

**Faerie Gold.** Without unique *ρ* values a payer could send two notes with the same nullifier, both of which look valid to the recipient although only one can ever be spent, an attack named Faerie Gold. Orchard rules the attack out by setting each new note's *ρ* to the nullifier of the note spent in the same Action, and since each nullifier appears on the chain at most once, every *ρ* is unique.

**Pool balances.** Nullifiers protect each note, and pool balances protect the supply as a whole. Each chain value pool keeps a running balance of the value the pool has received minus the value paid out, and a block which would make any pool's balance negative is invalid. The rule works like a turnstile at the entrance of each pool, so a bug in a shielded pool could create false value inside the pool, but no more value could leave the pool than entered (ZIP 209). The total supply across all pools must never exceed 21 million ZEC, and from NU7 the balance the Network Sustainability Mechanism holds for reissue must also stay non-negative.

## 4.8 What the proofs prove

Each proof has public inputs, which appear in the transaction, and a witness, which stays with the prover, and a node verifying the proof learns nothing beyond the checks below passing.

A spend proof shows the spent note's commitment is in the tree under the anchor, the nullifier is the right one for the note, and the randomized key *rk* disguises the owner's key. The proof also shows the value commitment hides the note's value and the address of the note comes from the same keys, and only the tree check is waived for a note of zero value. An output proof shows the new note's commitment and value commitment are made correctly.

An Orchard Action proves both sides at once, sets the new note's *ρ* to the spent note's nullifier, and lets a side whose flag is off carry only a dummy note. From NU6.3, when cross-address payments are disabled, the proof also requires the new note to go to the same address as the spent note. The Sprout proof checks the same kinds of facts for two notes in and two out, together with the balance and the tie between the spent notes and the transaction's signing key.

## 4.9 Encrypting and finding notes

The recipient took no part in making the transaction, so the note has to reach the recipient through the block chain. In Sapling and Orchard the sender makes a fresh ephemeral key for every note, combines the ephemeral key with the transmission key of the address to reach a shared secret, and encrypts the note under a key made from the secret. Sprout shares one ephemeral key between the two new notes of a JoinSplit. A second, shorter ciphertext lets the sender's outgoing viewing key read the payment later, and a sender who chose no outgoing viewing key fills the second ciphertext with an encryption under a random key nobody holds.

**Trial decryption.** The recipient's wallet tries every output on the chain with the incoming viewing key, like trying one key in every locked mailbox, and keeps a note only when the note rebuilt from the decrypted values matches the note commitment on the chain. The rebuilt note shows what arrived but not who sent the note, because the proof does not check the ciphertext and a note carries no sender address. Since Canopy the recipient also uses *rseed* to check the sender built the note and the ephemeral key correctly (ZIP 212).

Wallets must use the ephemeral key bytes exactly as the bytes appear on the chain. A wallet may also read transactions still waiting in the mempool, but must treat what the wallet learns as provisional and private, and must not assume the transaction will be mined.

**Scanning.** To tell whether a received note has been spent, the wallet computes the note's nullifier and watches for the nullifier in later spends. Sapling and Orchard wallets can scan with a full viewing key alone, while Sprout needs the spending key.
