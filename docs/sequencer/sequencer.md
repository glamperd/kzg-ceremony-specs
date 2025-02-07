# Sequencer

The primary job of the sequencer is to sequence participants in the ceremony by acting as the point of contact for participants. The sequencer is required to have a DoS-resilient internet connection and sufficiently powerful hardware to verify the transcript rapidly so that they are able to send the transcript to the next participant as soon as possible. Furthermore, the sequencer should be a semi-trusted entity as they have the power to censor participants, although they cannot affect the safety of the setup.


## BatchTranscript object

### `Witness`

```python
@dataclass
class Witness:
    running_products: List[bls.G1Point]
    pot_pubkeys: List[bls.G2Point]
```

### `Transcript`

```python
@dataclass
class Transcript:
    num_g1_powers: int
    num_g2_powers: int
    powers_of_tau: PowersOfTau  # as defined in ./README.md
    witness: Witness
```

### `BatchTranscript`

```python
@dataclass
class Transcript
    transcripts: List[Transcript]
```

## Verification

Upon receiving the transcript back from a participant, the Sequencer MUST perform the following checks.

### Contribution schema structure:

- __Schema Check__ - Verify that the received `contribution.json` matches the `contributionSchema.json` schema.
```python
def schema_check(contribution_json: str, schema_path: str) -> bool:
    with open(schema_path) as schema_file:
        schema = json.load(schema_file)
        try:
            jsonschema.validate(contribution_json, schema)
            return True
        except Exception:
            pass
    return False
```


### Contribution parameter check

This check verifies that the number of $\mathbb{G}_1$ and $\mathbb{G}_2$ powers meets expectations.

```python
def parameter_check(contribution: BatchContribution, transcript: BatchTranscript) -> bool:
    for (sub_contribution, sub_transcript) in zip(contribution.contributions, transcript.transcripts):
        if sub_contribution.num_g1_powers != sub_transcript.num_g1_powers:
            return False
        if sub_contribution.num_g2_powers != sub_transcript.num_g2_powers:
            return False
        if sub_contribution.num_g1_powers != len(sub_contribution.powers_of_tau.g1_powers):
            return False
        if sub_contribution.num_g2_powers != len(sub_contribution.powers_of_tau.g2_powers):
            return False
    return True
```


### Point Checks

- Prime Subgroup checks
    - __G1 Powers Subgroup check__ - For each of the $\mathbb{G}_1$ Powers of Tau (`g1_powers`), verify that they are actually elements of the prime-ordered subgroup.
    - __G2 Powers Subgroup check__ - For each of the $\mathbb{G}_2$ Powers of Tau (`g2_powers`), verify that they are actually elements of the prime-ordered subgroup.
    - __PoTPubkey Subgroup checks__ - Check that the PoTPubkey is actually an element of the prime-ordered subgroup.

```python
def subgroup_checks(contribution: BatchContribution) -> bool:
    for sub_contribution in contribution.contributions:
        if not all(bls.G1.is_in_prime_subgroup(P) for P in sub_contribution.powers_of_tau.g1_powers):
            return False
        if not all(bls.G2.is_in_prime_subgroup(P) for P in sub_contribution.powers_of_tau.g2_powers):
            return False
        if not bls.G2.is_in_prime_subgroup(sub_contribution.pot_pubkey):
            return False
    return True
```

- __Non-zero check__ - Check that the `pot_pubkey`s are not equal to the point at infinity.
```python
def non_zero_check(contribution: BatchContribution) -> bool:
    for sub_contribution in contribution.contributions:
        if bls.G2.is_inf(sub_contribution.pot_pubkey):
            return False
    return True
```

### Pairing Checks

_Note:_ The following pairing checks SHOULD be batched via a random linear combination which would reduce the number of final exponentiations to 2 and decrease the number of Miller loops needed.

- __Tau update construction__ - Verify that the latest contribution is correctly built on top of the previous ones. Let $[\tau^1_{k}]_1$ be the first power of tau from the most recent ($k$-th)`contribution` and $[\pi]_1^{k-1} := [\tau^1_{k-1}]_1$ be the transcript after updating from the previous correct `contribution`.  Verifying that $\tau$ was updated correctly can then be performed by checking $e([\pi_{k-1}]_1, [x_k]_2) \stackrel{?}{=}e([\pi_{k}]_1, g_2)$

```python
def tau_update_check(contribution: BatchContribution, transcript: BatchTranscript) -> bool:
    for (sub_contribution, sub_transcript) in zip(contribution.contributions, transcript.transcripts):
        tau = sub_contribution.powers_of_tau.g1_powers[1]
        transcript_product = sub_transcript.witness.running_products
        pk = sub_contribution.pot_pubkey
        if bls.pairing(transcript_product, pk) != bls.pairing(tau, bls.G2.g2):
                return False
    return True
```

- __Correct construction of G1 Powers__ - Verify that the $\mathbb{G}_1$ points provided are indeed incremental powers of (the same) $\tau$. This check is done by asserting that the next $\mathbb{G}_1$ point is the result of multiplying the previous one with $\tau$: $e([\tau^{i + 1}]_1, g_2) \stackrel{?}{=}e([\tau^i]_1, [\tau]_2)$. Note that the check that the $\tau$ in $[\tau]_2$ is the same as $\pi_k$ is done via the `g2_powers_check()` below which verifies that $e([\tau^i]_1, g_2) \stackrel{?}{=}e(g_2, [\tau^i]_2)$ and $[\tau^i]_1 = [\pi_k]_1$ due to the `Transcript` updates rules.

```python
def g1_powers_check(contribution: BatchContribution) -> bool:
    for sub_contribution in contribution.contributions:
        powers = sub_contribution.powers_of_tau.g1_powers
        pi = sub_contribution.powers_of_tau.g2_powers[1]
        for power, next_power in zip(powers[:-1], powers[1:]):
            if bls.pairing(next_power, bls.G2.g2) != bls.pairing(power, pi):
                return False
    return True
```

- __Correct construction of G2 Powers__ - Verify that the $\mathbb{G}_2$ points provided are indeed incremental powers of $\tau$ and that $\tau$ is the same across $\mathbb{G}_1$ and $\mathbb{G}_2$. This check is done by verifying that $\tau^i$ is the same across $[\tau^i]_1$ and $[\tau^i]_2$. $e([\tau^i]_1, g_2) \stackrel{?}{=}e(g_2, [\tau^i]_2)$

```python
def g2_powers_check(transcript: BatchTranscript) -> bool:
    for sub_contribution in transcript.contributions:
        g1_powers = sub_contribution.powers_of_tau.g1_powers
        g2_powers = sub_contribution.powers_of_tau.g2_powers
        for g1_power, g2_power in zip(g1_powers, g2_powers):
            if bls.pairing(bls.G1.g1, g2_power) != bls.pairing(g1_power, bls.G2.g2):
                return False
    return True
```

## Updating the transcript

Once the sequencer has performed the above Verification checks, they MUST update the transcript.


```python
def update_transcript(old_transcript: BatchTranscript, contribution: BatchContribution) -> BatchTranscript:
    new_transcript = Transcript(transcripts=[])
    for (sub_transcript, sub_ceremony) in zip(old_transcript.transcripts, contribution.ceremonies):
        new_sub_transcript = SubTranscript(
            num_g1_powers=sub_transcript.num_g1_powers,
            num_g2_powers=sub_transcript.num_g2_powers,
            powers_of_tau=sub_ceremony.powers_of_tau,
            witness=Witness(
                running_products=sub_transcript.witness.running_products + [sub_ceremony.powers_of_tau.g1_powers[1]],
                pot_pubkeys=sub_transcript.witness.pot_pubkeys + [sub_ceremony.pot_pubkey],
            )
        )
```

## Generating the contribution file for the next participant

Once the transcript has been updated, the sequencer MUST get a new `Contribution` file to send to the next participant.

```python
def get_new_contribution_file(transcript: BatchTranscript) -> BatchContribution:
    contribution = Contribution(contribution=[])
    for sub_transcript in transcript.transcripts:
        contribution.contributions.append(
            Contribution(
                num_g1_powers=sub_transcript.num_g1_powers,
                num_g2_powers=sub_transcript.num_g2_powers,
                powers_of_tau=sub_transcript.powers_of_tau,
                pot_pubkey='',
            )
        )
```

